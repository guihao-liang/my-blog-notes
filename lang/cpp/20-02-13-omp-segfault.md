---
title: don't let exception slipt away from omp
subtitle: C++ runtime abort when exception is uncaught
---

```cpp
(lldb) target create "lambda_omp_test.cxxtest"
Current executable set to 'lambda_omp_test.cxxtest' (x86_64).
(lldb) r
Process 18254 launched: '/Users/guihaoliang/Work/gui-1-36/debug/test/parallel/lambda_omp_test.cxxtest' (x86_64)
Running 5 test cases...
----------------------------------------------------------
40: 4040: : 40: 40: 40: 102334155
102334155
102334155
102334155
102334155
102334155
libc++abi.dylib: terminating with uncaught exception of type char const*libc++abi.dylib: libc++abi.dylib: terminating with uncaught exception of type char const*terminating with uncaught exception of type char const*libc++abi.dylib: libc++abi.dylib: libc++abi.dylib: terminating with uncaught exception of type char const*
libc++abi.dylib: terminating with uncaught exception of type char const*
libc++abi.dylib: terminating with uncaught exception of type char const*
libc++abi.dylib: terminating with uncaught exception of type char const*

libc++abi.dylib: terminating with uncaught exception of type char const*

terminating with uncaught exception of type char const*
terminating with uncaught exception of type char const*
libc++abi.dylib: terminating with uncaught exception of type char const*Process 18254 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGABRT
    frame #0: 0x00007fff5ca882c6 libsystem_kernel.dylib`__pthread_kill + 10
libsystem_kernel.dylib`__pthread_kill:
->  0x7fff5ca882c6 <+10>: jae    0x7fff5ca882d0            ; <+20>
    0x7fff5ca882c8 <+12>: movq   %rax, %rdi
    0x7fff5ca882cb <+15>: jmp    0x7fff5ca82457            ; cerror_nocancel
    0x7fff5ca882d0 <+20>: retq
Target 0: (lambda_omp_test.cxxtest) stopped.
(lldb) up
frame #1: 0x00007fff5cb43bf1 libsystem_pthread.dylib`pthread_kill + 284
libsystem_pthread.dylib`pthread_kill:
->  0x7fff5cb43bf1 <+284>: movl   %eax, %r15d
    0x7fff5cb43bf4 <+287>: cmpl   $-0x1, %eax
    0x7fff5cb43bf7 <+290>: jne    0x7fff5cb43c23            ; <+334>
    0x7fff5cb43bf9 <+292>: callq  0x7fff5cb46afe            ; symbol stub for: __error
(lldb) up
frame #2: 0x00007fff5c9f26a6 libsystem_c.dylib`abort + 127
libsystem_c.dylib`abort:
->  0x7fff5c9f26a6 <+127>: movl   $0x2710, %edi             ; imm = 0x2710
    0x7fff5c9f26ab <+132>: callq  0x7fff5c9c5568            ; usleep$NOCANCEL
    0x7fff5c9f26b0 <+137>: callq  0x7fff5c9f26b5            ; __abort

libsystem_c.dylib`__abort:
    0x7fff5c9f26b5 <+0>:   cmpq   $0x0, 0x36353acb(%rip)    ; gCRAnnotations + 7
(lldb)
frame #3: 0x00007fff5972b641 libc++abi.dylib`abort_message + 231
libc++abi.dylib`__cxa_bad_cast:
->  0x7fff5972b641 <+0>: pushq  %rbp
    0x7fff5972b642 <+1>: movq   %rsp, %rbp
    0x7fff5972b645 <+4>: pushq  %rbx
    0x7fff5972b646 <+5>: pushq  %rax
(lldb)
frame #4: 0x00007fff5972b7df libc++abi.dylib`default_terminate_handler() + 267
libc++abi.dylib`default_unexpected_handler:
->  0x7fff5972b7df <+0>:  pushq  %rbp
    0x7fff5972b7e0 <+1>:  movq   %rsp, %rbp
    0x7fff5972b7e3 <+4>:  leaq   0xd23f(%rip), %rax        ; "unexpected"
    0x7fff5972b7ea <+11>: movq   %rax, 0x38d5d6ff(%rip)    ; cause
(lldb)
frame #5: 0x00007fff5b181eeb libobjc.A.dylib`_objc_terminate() + 105
libobjc.A.dylib`_objc_terminate:
->  0x7fff5b181eeb <+105>: addq   $0x8, %rsp
    0x7fff5b181eef <+109>: popq   %rbx
    0x7fff5b181ef0 <+110>: popq   %rbp
frame #5: 0x00007fff5b181eeb libobjc.A.dylib`_objc_terminate() + 105
libobjc.A.dylib`_objc_terminate:
->  0x7fff5b181eeb <+105>: addq   $0x8, %rsp
    0x7fff5b181eef <+109>: popq   %rbx
    0x7fff5b181ef0 <+110>: popq   %rbp
    0x7fff5b181ef1 <+111>: jmp    0x7fff5b17fe3f            ; objc_end_catch
(lldb)
frame #6: 0x00007fff5973719e libc++abi.dylib`std::__terminate(void (*)()) + 8
libc++abi.dylib`std::__terminate:
->  0x7fff5973719e <+8>:  leaq   0x2413(%rip), %rdi        ; "terminate_handler unexpectedly returned"
    0x7fff597371a5 <+15>: xorl   %eax, %eax
    0x7fff597371a7 <+17>: callq  0x7fff5972b55a            ; abort_message
    0x7fff597371ac <+22>: ud2
(lldb)
frame #7: 0x00007fff59737213 libc++abi.dylib`std::terminate() + 51
libc++abi.dylib`std::terminate:
->  0x7fff59737213 <+51>: mfence
    0x7fff59737216 <+54>: leaq   0x38d51cc3(%rip), %rax    ; __cxa_terminate_handler
    0x7fff5973721d <+61>: movq   (%rax), %rdi
    0x7fff59737220 <+64>: callq  0x7fff59737196            ; std::__terminate(void (*)())
(lldb)
frame #8: 0x0000000100624aff libunity_shared_for_testing.dylib`__clang_call_terminate + 15
libunity_shared_for_testing.dylib`__clang_call_terminate:
->  0x100624aff <+15>: nop

libunity_shared_for_testing.dylib`std::__1::char_traits<char>::assign:
    0x100624b00 <+0>:  pushq  %rbp
    0x100624b01 <+1>:  movq   %rsp, %rbp
    0x100624b04 <+4>:  movq   %rdi, -0x8(%rbp)
(lldb)
frame #9: 0x0000000105606c89 libunity_shared_for_testing.dylib`::.omp_outlined._debug__(.global_tid.=0x00007ffeefbf9ec0, .bound_tid.=0x00007ffeefbf9eb8, fn=0x00007ffeefbfa4d0)> &) at lambda_omp.cpp:21:23
   18   // start a team of threads
   19   #pragma omp parallel default(none)
   20     {
-> 21       size_t nworkers = omp_get_num_threads();
   22   #pragma omp for schedule(guided)
   23       for (size_t ii = 0; ii < nworkers; ii++) {
   24         fn(ii, nworkers);
(lldb) up
frame #10: 0x0000000105606d25 libunity_shared_for_testing.dylib`::.omp_outlined.(.global_tid.=0x00007ffeefbf9ec0, .bound_tid.=0x00007ffeefbf9eb8, fn=0x00007ffeefbfa4d0) at lambda_omp.cpp:20:3
   17   #if defined(_OPENMP) && !defined(DISABLE_OPENMP)
   18   // start a team of threads
   19   #pragma omp parallel default(none)
-> 20     {
   21       size_t nworkers = omp_get_num_threads();
   22   #pragma omp for schedule(guided)
   23       for (size_t ii = 0; ii < nworkers; ii++) {
(lldb)
frame #11: 0x00000001057bc643 libunity_shared_for_testing.dylib`__kmp_invoke_microtask at z_Linux_asm.S:1151
   1148         cmovnsq (%r11), %rdx    // p_argv[0] -> %rdx (store 3rd parm to pkfn)
   1149 #endif // KMP_MIC
   1150
-> 1151         call    *%rbx           // call (*pkfn)();
   1152         movq    $1, %rax        // move 1 into return register;
   1153
   1154         movq    -8(%rbp), %rbx  // restore %rbx using %rbp since %rsp was modified
```