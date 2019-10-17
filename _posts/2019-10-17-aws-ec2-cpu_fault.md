---
layout: post
title:  "AWS EC2 HVM instances may lie about cpuid_fault"
date:   2019-10-17
categories: Linux
---

cpuid faulting is a feature made available after intel ivy bridge CPUs, Linux kernel support have been pushed by `rr`,
via `arch_prctl(ARCH_SET_CPUID)`[1]. It allows user space programs simulate CPUID instruction, by trapping cpuid instruction to SEGSEGV.

This is done by query `PLATFORM_INFO` (0xce) MSR, bit-31. When bit-31 is set, `cpuid_fault` can be done by write/clear `MISC_FEATURES_ENABLES`, bit-0.

On an AWS EC2 t2.micro (virtualized) instance, `cpuid_fault` is present:

```
$ sudo modprobe msr
$ sudo rdmsr -x 0xce
20080c33f3811800      # bit-31 is set.
And can be confirmed by cat /proc/cpuinfo as well:

$ cat /proc/cpuinfo | grep cpuid_fault
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm cpuid_fault invpcid_single pti fsgsbase bmi1 avx2 smep bmi2 erms invpcid xsaveopt
```

However, below program doesn't work as expected:

```c
#include <sys/types.h>
#include <sys/syscall.h>
#include <sys/prctl.h>
#include <asm/prctl.h>
#include <assert.h>
#include <unistd.h>
#include <stdio.h>
#include <cpuid.h>
#include <signal.h>
#include <string.h>

int arch_prctl(int code, unsigned long addr);

static void sigsegv_action(int signo, siginfo_t* siginfo, void* ucontext) {
  const char msg[] = "got expected sigsegv\n";
  write(STDOUT_FILENO, msg, sizeof(msg));
  _exit(0);
}

int main(int argc, char* argv[])
{
  assert(arch_prctl(ARCH_SET_CPUID, 0) == 0);

  unsigned long eax, ebx, ecx, edx;

  struct sigaction sa;
  memset(&sa, 0, sizeof(sa));

  sa.sa_flags = SA_ONESHOT | SA_RESTART | SA_SIGINFO | SA_RESETHAND;
  sa.sa_sigaction = sigsegv_action;
  assert(sigaction(SIGSEGV, &sa, NULL) == 0);

  // should segfault
  __cpuid(0x1, eax, ebx, ecx, edx);
  printf("eax=%lx\n", eax);

  return 0;
}
```

The expected result is it should segfault (print got expected sigsegv), but on `t2.micro`, the program prints `eax=306f2` instead. It implies the kernel indeed received the `arch_prctl` request, and set the hardware register, but the HVM (`nitro`??) somehow ignored the write to `MISC_FEATURES_ENABLES`, bit-0.

More details can be found in this[2] application note.

At first I thought it might be an AWS kernel (optimization) issue, hence tried both on kernel: 4.15.0-1051-aws and 4.15.0-65-generic, but the results were the same.

More information about the system setup:

AWS ec2 t2.micro
stock ubuntu 18.04

[1] [Add arch_prctl for controlling CPUID instruction](https://lwn.net/Articles/713970/)

[2] [virtualization technology flexmigration application note](https://www.intel.com/content/dam/www/public/us/en/documents/application-notes/virtualization-technology-flexmigration-application-note.pdf)
