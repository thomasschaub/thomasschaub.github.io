---
layout: post
title:  "Code injection using LD_PRELOAD"
date:   2019-02-18 19:56:47 +0100
categories: jekyll update
---
On a typical desktop Linux system, the majority of the programs link dynamically against one or more libraries.
The `ldd` command prints the list of shared libraries required by an application.
Using `ldd` to examine the `ls` command on Ubuntu 18.04 yields:

```
$ ldd /bin/ls
	linux-vdso.so.1 (0x00007ffc1a0d1000)
	libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007f3509ea5000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3509ab4000)
	libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3 (0x00007f3509842000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f350963e000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f350a2ef000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f350941f000)
```

We see that `ls` links against, for example, the C standard library (`libc.so.6`).
It is unclear which functions from the C standard library are used exactly, but it is pretty safe to assume that `ls` uses `malloc`.

When you run a dynamically linked executable (such as `ls`), upon process creation, the operating system dispatches to a dynamic linker, usually `ld-linux.so.2`, .
The dynamic linkers locates and loads shared libraries and executes the program afterwards.
When `ls` calls `malloc`, it is the dynamic linker's job to resolve the symbol.

Interestingly, there is nothing special about `ld-linux.so.2`.
It gets selected by the OS, because `ls` defines it in an ELF program header.
At the end of the day, the dyanmic linker is just another program.
Like many other programs, `ld-linux.so.2` has configuration options.
In this post I want to highlight the `LD_PRELOAD` option.

If `LD_PRELOAD` is set to a shared object path, the dynamic linker will load the given library and consider it during symbol resolution.
We can test this by injecting a library that prints something on each call to `malloc`.
To make our lives easier, we pretend that we ran out of memory and return `NULL`.

Here's an implementation:

```
void* malloc(size_t size)
{
  write(STDOUT_FILENO, "*** malloc\n", 11);
  return NULL;
}
```

If we create a dynamic library `libfoo.so` from this function, we can inject our `malloc` using `LD_PRELOAD`:

```
$ LD_PRELOAD=./libfoo.so ls
*** malloc
...
*** malloc
ls: memory exhausted
```

We see that `ls` calls `malloc` a few times, but eventually gives up, thinking that we're out of memory.
This is exactly what our `malloc` function suggested by returning `NULL`.

Speaking of memory, we need to be careful not to call `malloc` (also not indirectly) from our replacement.
This can happen, for example, when calling `printf`, hence we use a plain `write`.

At this point we have successfully told the dynamic linker to use our code instead of the code it would use normally.
However, instead of completely replacing a function, we might want to wrap it instead.
For our example, we can print a message and then forward the request to the original `malloc`.
To obtain the address of the original function, we use `dlsym`.

```
typedef void*(*malloc_t)(size_t);

void* malloc(size_t size)
{
  write(STDOUT_FILENO, "*** malloc\n", 11);
  malloc_t f = (malloc_t)dlsym(RTLD_NEXT, "malloc");
  f(size);
}
```

Note that by using `RTLD_NEXT`, we make sure that `dlsym` will not return our own address, but the address of the next library in library search order.
The [dlsym manpage][man-dlsym] specifically mentions this as being useful for writing wrappers.

If we run `ls` while preloading the updated function, we get

```
$ LD_PRELOAD=./libbar.so ls
*** malloc
[...]
*** malloc
CMakeCache.txt	CMakeFiles  cmake_install.cmake  libbar.so  libfoo.so  Makefile
```

Note that `ls` no longer complains about exhausted memory, but rather prints the contents of the demo repo's build directory.

The technique in this post is used by, for example, [RenderDoc][render-doc], a renderer debugger.
RenderDoc captures and serializes all calls a given program makes to a graphics API.
A capture can later be replayed and analyzed.
Recording is implemented by using `LD_PRELOAD` to inject `librenderdoc.so`, which provides wrappers for all major graphics APIs.

[man-dlsym]:   https://linux.die.net/man/3/dlsym
[render-doc]:  https://renderdoc.org