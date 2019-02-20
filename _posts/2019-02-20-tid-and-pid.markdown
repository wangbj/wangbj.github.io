---
layout: post
title:  "tid, pid, ppid and pgid in a threaded context"
date:   2018-02-20
categories: Linux
---

- `tid`: (see `man 2 gettid`) task id
- `pid`: process id (returned by `getpid()`)
- `ppid`: parent pid (`getppid()`)
- `pgid`: return process group id (`getpgid(_)`)

`pid` could be confusing in a multi-threaded program in Linux (NPTL). multiple threads can share the same `pid`.
The kernel, however, views them as `tid`, since `NPTL` is implemented in user space, the kernel doesn't really know
much about `NPTL` threads, but threads created (`clone` with `CLONE_THREAD` flag) in the same process share the same
`pid`.

Their relations seems to be:

For reguar processes (not threads):

  - `tid` == `pid`

For threads:

  - each thread has their own `tid` (`tid` is also unique, but same as `pid`, they do wraps, see `cat /proc/sys/kernel/pid_max`).
  - threads created in the same process, share the same `pid` (not `pgid`)

To demostrate the problem, we can first do a `fork`, and then create threads (`pthread_create`) in both parent and child process.
The result shows:

  - threads group 1 (parent): all threads have different `tid`, share the same `pid` as parent, `ppid` = `parent.ppid`, `pgid` = `parent`
  - threads group 2 (child):  all threads have different `tid`, share the same `pid` as child, `ppid` = `parent`, `pgid` = `parent`

So even though the `pgid` are indeed the same for all threads created above, it is the `pid` from above should be used to identify their
relationship with the parent. This can be revealed by check `/proc/pid/task/tid`, i.e.: The `tid` can be discovered under `/proc/pid`.

Example source code can be found in [1]:

[1]: [source code](https://gist.github.com/wangbj/b54af0a574f3b043194dce249c874ef6)
