---
layout: post
title:  "Linux ptrace and exit_group"
date:   2019-03-13
categories: Linux
---

`exit_group` is a Linux syscall to kill all processes in the same process group, 
by *kill*, it means kernel sends `SIGKILL` to all processes in the same process group.
Assuming you're writing a non-trival tracer with proper thread support, `exit_group`
 can be quite difficult to deal with. According to `strace` document found 
[here](https://github.com/strace/strace/blob/e0f0071b36215de8a592bf41ec007a794b550d45/README-linux-ptrace#L64)

```
If PTRACE_O_TRACEEXIT option is on, PTRACE_EVENT_EXIT will happen
before actual death. This applies to exits on exit syscall, group_exit
syscall, signal deaths (except SIGKILL), and when threads are torn down
on execve in multi-threaded process.

Tracer cannot assume that ptrace-stopped tracee exists. There are many
scenarios when tracee may die while stopped (such as SIGKILL).
Therefore, tracer must always be prepared to handle ESRCH error on any
ptrace operation. Unfortunately, the same error is returned if tracee
exists but is not ptrace-stopped (for commands which require stopped
tracee), or if it is not traced by process which issued ptrace call.
Tracer needs to keep track of stopped/running state, and interpret
ESRCH as "tracee died unexpectedly" only if it knows that tracee has
been observed to enter ptrace-stop. Note that there is no guarantee
that waitpid(WNOHANG) will reliably report tracee's death status if
ptrace operation returned ESRCH. waitpid(WNOHANG) may return 0 instead.
IOW: tracee may be "not yet fully dead" but already refusing ptrace ops.

Tracer can not assume that tracee ALWAYS ends its life by reporting
WIFEXITED(status) or WIFSIGNALED(status).
```

It appears quite terrifying when reading above text, and in practice it could
be even more devastating: the group exit (caused by `exit_group` syscall) can
happen anytime: say if task A called `exit_group`, while task B in the same
process group caught a ptrace event, for instance `PTRACE_EVENT_SECCOMP`, which
happens very often when `seccomp` is used. task B could be stopped because of 
seccomp event, most of the time the tracer would at least check the registers 
for seccomp, by `PTRACE_GETREGS`, however, it could unexpectedly fail because 
of task A's `exit_group`. And that's just one case, the failure (with `-ESRCH`) 
could happen pretty much any ptrace operations, making the tracer needlessly 
verbose.

The linked document also suggests the tracer have no way to figure out whether 
or not ptrace failure with `-ESRCH` is caused by dying of tracee, or simply
because of tracee is running. It wouldn't be too much trouble to keep track of 
tracees' status, after all, that's one task a tracer intended to do. However, 
When we tried to deal with this case, something even more interesting 
happened: 

  * task B received a `PTRACE_EVENT_SECCOMP`, it failed to do meanfully 
    job because `PTRACE_GETREGS` failed.
  * As suggested by the document, task B must have received `SIGKILL`, and
    was about to exit, or have exited already.
  * for the safe (or curiosity) side, we also tried to check `/proc/<tid>/stat`
    To see if the entry exists at all.
  * the proc entry did exist, with state set to `t`, meaning it entered ptrace
    stop
  * after above steps, if `PTRACE_GETREGS` was called again, it did succeed 2nd
    time;
  * If `waitpid` was called, it shows task B entered `PTRACE_EVENT_EXIT`

It seems like as long as a task (B) received `SIGKILL`, any ptrace operations 
would fail immediately; however, because `PTRACE_EVENT_EXIT` ( assume 
`PTRACE_O_TRACEEXIT` is set); tracer could have received the event exit. And in 
the event exit stop, ptrace operations becomes available again. It's quite 
frustrating to discover such a confusing phenomenal, and indeed, it is documented
in ptrace manual, section **BUGS**:

```
BUGS
	   // snip

       A SIGKILL signal may still cause a PTRACE_EVENT_EXIT stop before actual signal death.  This  may  be  changed  in  the
       future; SIGKILL is meant to always immediately kill tasks even under ptrace.  Last confirmed on Linux 3.13.
```

Why it matters? you might say. Well, if that happens, we can not simply *assume* 
the tracee just exited, because it might simply in `PTRACE_EVENT_EXIT` stop, and the 
tracer must *resume* it, by either `PTRACE_CONT`, or `PTRACE_DETACH`, or whatever you 
would do with ptrace event exit. Even there might be a *resume* from previous ptrace 
event, such as from `PTRACE_EVENT_SECCOMP`, it might have failed, as explained earlier.

Blow is a not quite useful log for this issue, it seems quite easy to run into when 
running even the simpliest go programs:

Here a task in a process group called `exit_group`:

```
17667 seccomp syscall SYS_exit_group@4520ab, hook: None, preloaded: false
```

And another task got a `SECCOMP` event:
```
[sched] 17671 Ok(PtraceEvent(Pid(17671), SIGTRAP, 7))
```

Tracer (seccomp handler) failed to `PTRACE_GETREGS`:
```
17671 getregs returned: Sys(ESRCH)      // this ptrace getregs failed.
```

Assumes *17671* is about to eixt:
```
[sched] 17671 failed to run, assuming killed
```

And it is not dead by read `/proc/17671/stat`:
```
[sched] task 17671 refused to be traced while alive, stat: 17671 (conftest) t 17666 17666 31597 34816 17666 1073742912 267 0 0 0 0 0 0 0 20 0 5 0 8544029 4300800 480 18446744073709551615 4194304 5322272 140737488348096 0 0 0 0 3145728 2143420159 0 0 0 -1 1 0 0 0 0 0 6361088 6453568 6582272 140737488348693 140737488348704 140737488348704 140737488351213 1541
```

And `PTRACE_GETREGS` now succeed (2nd time):
```
rip = Ok(452703)                // this ptrace getregs succeeded
```

call `waitpid(17671,...)` indicate `17671` was in ptrace stop, which is
consistent with `/proc/17671/stat`:
```
17671 Ok(PtraceEvent(Pid(17671), SIGTRAP, 6))
```
