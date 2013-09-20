fsyscall
========

Inline system call wrapper for the crazy people


Why?
====

Very simple - I would like to have my program use the `syscall` instruction (x86-64)
directly, instead of calling a glibc wrapper function.


Example
=======

"Standard" code snippet:


    int mkchdir(const char *buf) {
            int rc = chdir(buf);
            if ((rc != 0) && (errno == ENOENT)) {
                    mkdir(buf, 0666);
                    rc = chdir(buf);
            }
            return (rc == 0);
    }

My code snippet:

    int mkchdir(const char *buf) {
            int rc = fsys_chdir(buf);
            if ((rc != 0) && fsys_errno(rc, ENOENT)) {
                    fsys_mkdir(buf, 0666);
                    rc = fsys_chdir(buf);
            }
            return (rc == 0);
    }


Compare their assembly code (both x86-64):

    0000000000000000 <mkchdir>:
       0:   53                      push   %rbx
       1:   31 c0                   xor    %eax,%eax
       3:   48 89 fb                mov    %rdi,%rbx
       6:   e8 00 00 00 00          callq  b <mkchdir+0xb>
                            7: R_X86_64_PC32        chdir-0x4
       b:   85 c0                   test   %eax,%eax
       d:   ba 01 00 00 00          mov    $0x1,%edx
      12:   74 0c                   je     20 <mkchdir+0x20>
      14:   e8 00 00 00 00          callq  19 <mkchdir+0x19>
                            15: R_X86_64_PC32       __errno_location-0x4
      19:   31 d2                   xor    %edx,%edx
      1b:   83 38 02                cmpl   $0x2,(%rax)
      1e:   74 08                   je     28 <mkchdir+0x28>
      20:   89 d0                   mov    %edx,%eax
      22:   5b                      pop    %rbx
      23:   c3                      retq
      24:   0f 1f 40 00             nopl   0x0(%rax)
      28:   48 89 df                mov    %rbx,%rdi
      2b:   be b6 01 00 00          mov    $0x1b6,%esi
      30:   31 c0                   xor    %eax,%eax
      32:   e8 00 00 00 00          callq  37 <mkchdir+0x37>
                            33: R_X86_64_PC32       mkdir-0x4
      37:   48 89 df                mov    %rbx,%rdi
      3a:   31 c0                   xor    %eax,%eax
      3c:   e8 00 00 00 00          callq  41 <mkchdir+0x41>
                            3d: R_X86_64_PC32       chdir-0x4
      41:   31 d2                   xor    %edx,%edx
      43:   85 c0                   test   %eax,%eax
      45:   0f 94 c2                sete   %dl
      48:   89 d0                   mov    %edx,%eax
      4a:   5b                      pop    %rbx
      4b:   c3                      retq

vs.

    0000000000000000 <mkchdir>:
       0:   b8 50 00 00 00          mov    $0x50,%eax    # __NR_chdir
       5:   0f 05                   syscall
       7:   83 f8 fe                cmp    $0xfffffffe,%eax
       a:   75 13                   jne    1f <mkchdir+0x1f>
       c:   be b6 01 00 00          mov    $0x1b6,%esi
      11:   b8 53 00 00 00          mov    $0x53,%eax    # __NR_mkdir
      16:   0f 05                   syscall
      18:   b8 50 00 00 00          mov    $0x50,%eax    # __NR_chdir
      1d:   0f 05                   syscall
      1f:   85 c0                   test   %eax,%eax
      21:   0f 94 c0                sete   %al
      24:   0f b6 c0                movzbl %al,%eax
      27:   c3                      retq

So far, if you have realized why I did this, you are as crazy as me and you probably
want to have a try.  Otherwise, sorry for wasting your time :)

By the way, you probably need some knowledge on Linux system call ABI (at least for x86-64)
to use this.

APIs
====

In short: just prefix `fsys_` to a system call.

Some wrapper functions have modified prototypes or names, such as
[`open`](http://linux.die.net/man/2/open) maps to two functions:
`fsys_open2` (without the mode parameter) and `fsys_open3` (with the mode parameter).

`fsys_errno`, `fsys_errno_val`: Check error number.
`fsys_mmap_failed`: Check [`mmap`](http://linux.die.net/man/2/mmap) return value.


Pros and cons
=============

Pros:

* It makes paranoid people like me happy.

Cons:

* This wrapper only works for standard SysV/x86-64 and [x32](http://en.wikipedia.org/wiki/X32_ABI) ABIs.
  It has no effect on other platforms or ABIs.
  (NOTE: The x32 ABI is a new ABI for x86-64.  It is not the ABI used on 32-bit x86 systems.)
* The performance benefit is very very tiny.
* No supoprt for [cancellation points](http://stackoverflow.com/questions/433989/posix-cancellation-points)!
* Not all system calls are included.  Only the ones I often use are.
* When Linus releases new system calls and obsoletes some old ones,
  "standard" programs automatically take advantage of the new ones when glibc is upgraded,
  while program using fsyscall needs recompilation.
* `fsys_` functions fall back to glibc wrappers on platforms other than x86-64.
  They often have different header requirement than the inline versions, which causes
  pains in developing programs intended to be portable.

Disclaimer
==========

I am no fan of GNU or GPL, absolutely.
The sole reason why this repository is licensed under LGPL rather than a more permissive license
is that my code is partially based
on file `sysdeps/unix/sysv/linux/x86_64/sysdep.h` in [glibc](http://www.gnu.org/software/libc/).
