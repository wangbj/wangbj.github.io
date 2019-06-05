---
layout: post
title:  "seccomp multiple rules are useless"
date:   2019-06-05
categories: Linux
---

Both `prctl` and `seccomp` syscalls can be used to install seccomp(-bpf) filters, According to `seccomp(2)`:

```
              If fork(2) or clone(2) is allowed by the filter, any child processes will be constrained to the same system call filters as the parent.  If execve(2) is allowed, the existing filters will
              be preserved across a call to execve(2).
```

So it would be intuitive to have filters installed in parent process, if the application is a tracer/monitor used to launch arbitrary programs. That could become a problem when we need to inject new
seccomp rules: Linux allows multiple rules to be installed, and the most recent one installed get triggered first, with the priorities:

```
 SECCOMP_RET_KILL* > SECCOMP_RET_TRAP > SECCOMP_RET_ERRNO > SECCOMP_RET_TRACE > SECCOMP_RET_LOG > SECCOMP_RET_ALLOW
```

So if there're two rules, one set to `trap`, the other set to `allow`, the decision would be `trap`. It totally makes sense, however Linux doesn't have a way to uninstall seccomp filters, even
though it allows install new ones as mentioned. Also according the `seccomp(2)`:

```
       It is strongly recommended to use a whitelisting approach whenever possible because such an approach is more robust and simple.  A blacklist will have to be updated whenever a  potentially  dan‐
       gerous  system  call is added (or a dangerous flag or option if those are blacklisted), and it is often possible to alter the representation of a value without altering its meaning, leading to a
       blacklist bypass.
```

Using whitelist implies everything is filtered by a rule with higher priority, unless the whitelisted ones. i.e.: we can create a rule allow all syscalls to be `traced`, unless the syscall is originated
from a certain range. This however becomes problematic, if we decided we need to install 2nd (or more) rules later on: we might want to `allow` syscall from another range, but according to the priority
rules, it simply has no effects, because the new range is `traced` by the first rule. So it makes using multiple rules pretty much useless. Using backlist approach couldn't be any better, because we'll have
to keep track of all the ranges to be blacklisted, if the application loads tons of shared libraries, it becomes quite a headache, not to mention blacklisting is much less secure.

It would be nice if Linux allows replacing seccomp rules on the fly, i.e.: We can get the current seccomp rule (this is supported by `ptrace` -> `PTRACE_SECCOMP_GET_FILTER` since 4.4), edit (run-time) then
replace it, or allow user to uninstall previous seccomp rules. But this is not something supported as of today. library like `libseccomp` has function `seccomp_release`, but as its manage shows, Any
seccomp filters loaded into the kernel are not affected. Granted there're security risks to replace existing ones, but it can be mitigated (by checking the caller's priviledge); and security should not
sacrifice usability, but that's just my opinion.

