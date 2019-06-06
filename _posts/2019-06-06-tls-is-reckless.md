---
layout: post
title:  "tls is reckless"
date:   2019-06-06
categories: Linux
---

In *x86_64*, thread local storage (tls) is implemented in userspace, mostly in *libc*. However, with dynamically
 linked application the dynamic linker also must initialize tls for the application, most importantly, it exports
 function `__tls_get_addr` (`__tls_get_new` is an weak alias to this function). tls function related to *pthread*
 are often implemented in *libc* or *libpthread*.

Because `__tls_get_addr` is provided by dynamic linker, it has several implications:

  * writing a dynamic linker becomes much harder.
  
    `__tls_get_addr` usually returns some vector from `thread` structure, meaning it is completely implementation
    specific, *musl* and *glibc* implementations are totally non-interchangeable.
    
    Because the dynamic linker has to implements `__tls_get_addr`, which is somehow `(p)thread` related, to write a
    dynamic linker also implies you have to write a *libc* with *pthread*s, this increase the complexiy by an order
    of magnitude: see [This](https://github.com/m4b/dryad#todos). 
    
  * the dynamic linker is also a dynamic shared object (DSO) at the same time
  
    for *glibc*, `ld-linux.so` also exports several symbols, but they doesn't have ABI compatibily: if you're not *glibc*
    it is very unlikely you can get much sense of the exported symbols
    
  * you cannot really have two *libc*s linked/mapped in the same application.
  
    It may sound stupid to link two *libc*s in a single application, but it can happen, i.e.: you may create a freestanding
    DSO statically compiled with *musl* libc (but as a whole it is still a DSO!), and then use `LD_PRELOAD` to preload with
    an application linked against glibc. At first everything might seems fine, until you use tls: because the two *libc*s
    have completely different tls implimentations. This could be fine if the `LD_PRELOAD` DSO was written in low level
    language like C, for rust (*cdylib*) it is totally a different story - rust use tls extensively - pretty much any
    (mutable) global variables rust encourage use `thread_local` (tls), or use it implicitly so wen `println!` uses tls.
    
  * Application can be built in different ways.
  
    Application can link against *glibc*, or *musl* *libc*, or statically linked with them; or it can be a go application, 
    who is statically linked, but does not use tls at all.
    
    Say if we `LD_PRELOAD` a freestanding DSO (statically linked *musl* *libc*), with an application dynamically linked aginst
    *glibc*, it is somehow possible to force the former use the same `__tls_get_addr` in (*glibc*'s) `ld-linux.so`, however
    it is a super dirty hack, requires patching the freestanding DSO during runtime; and it is not easy to support so many 
    different configurations, let alone different architecture.

I really hope tls, at least `__tls_get_addr` is provided by Kernel, as `get_thread_area` from x86 (not available in x86_64), 
so that we don't have to go through different tls implementations; because `__tls_get_addr` from `%fs:<offset>`, it is 
unlikely tls can be virtualized, or allow trapping.
    
