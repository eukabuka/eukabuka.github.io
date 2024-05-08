# CFI icall analysis

This is meant to share my observations from playing around with a binary compiled with
Clang CFI. I'm mostly interested in how CFI can be used with C codebases, and will focus
on the `icall` mechanism intended to prevent certain forms of control flow hijacks
involving indirect control flow(eg: function pointer hijacks).

This [trailofbits blogpost](https://blog.trailofbits.com/2016/10/17/lets-talk-about-cfi-clang-edition/) describes that various kinds of CFI options. [This](https://github.com/trailofbits/clang-cfi-showcase) is an github repository with related source code meant to showcase the kinds of control flow that are allowed/blocked by CFI. I found disassembling
the cfi-icall example in this repository to be very educational.

Before proceeding, browse through [the icall example](https://github.com/trailofbits/clang-cfi-showcase/blob/master/cfi_icall.c). When [compiled with CFI icall](https://github.com/trailofbits/clang-cfi-showcase/blob/master/Makefile#L16), passing in command line
arguments `0`, or `1` WAI, and using arguments `2` or `3` result in a SIGILL. Without
the CFI options, using `2` and `3` would not result in a SIGILL.

(All examples are compiled with clang-14 on Ubuntu).

Let's quickly look at the relevant disassembly of the following source lines when no
CFI is used.

```C
    return f.int_funcs[idx](idx);
```

```bash
0x0000000000001292 <+322>:	lea    rax,[rip+0x2d9f]        # 0x4038 <f>
0x0000000000001299 <+329>:	mov    rax,QWORD PTR [rax+rcx*8]
0x000000000000129d <+333>:	mov    edi,DWORD PTR [rbp-0x14]
0x00000000000012a0 <+336>:	call   rax
```

The address of the object `f` is loaded into `rax`. Then the pointer at the offset `rcx` is read into `rcx`. The argument to the function is loaded into `edi` and the function
is then called.

Before looking at the CFI version, here is a preview of what to expect when a function
that is allowed by CFI(arguments `0` or `1`) is called:
* The address loaded at `rax+rcx*8` will not necessarily point to the function themselves
anymore. Instead they will point into an entry in a jumptable. This table will contain
`jmp` instructions to the actual function target.
* before calling into the jumptable entry, the index into the jumptable entry will be
validated. if the index is valid, the `call` instruction is executed. If not, SIGILL is
triggered.

The CFI version looks as follows:
```bash
0x000000000000128f <+319>:	lea    rcx,[rip+0x2da2]        # 0x4038 <f>
0x0000000000001296 <+326>:	mov    rcx,QWORD PTR [rcx+rax*8]
0x000000000000129a <+330>:	lea    rax,[rip+0x18f]        # 0x1430 <int_arg>
0x00000000000012a1 <+337>:	mov    rdx,rcx
0x00000000000012a4 <+340>:	sub    rdx,rax
0x00000000000012a7 <+343>:	mov    rax,rdx
0x00000000000012aa <+346>:	shr    rax,0x3
0x00000000000012ae <+350>:	shl    rdx,0x3d
0x00000000000012b2 <+354>:	or     rax,rdx
0x00000000000012b5 <+357>:	cmp    rax,0x2
0x00000000000012b9 <+361>:	jbe    0x12c0 <main+368>
0x00000000000012bb <+363>:	ud1    eax,DWORD PTR [eax+0x2]
0x00000000000012c0 <+368>:	mov    edi,DWORD PTR [rbp-0x8]
0x00000000000012c3 <+371>:	call   rcx
```

Lets break this down.

## Using arguments `0` or `1`
```bash
0x000000000000128f <+319>:	lea    rcx,[rip+0x2da2]        # 0x4038 <f>
0x0000000000001296 <+326>:	mov    rcx,QWORD PTR [rcx+rax*8]
```

This snippet loads an address into rcx. In a debugger(with CLI argument `0`), we can see that
`rcx` points to an entry in the jumptable. Each entry is 8 bytes(`5` bytes for the
jump + `3` padding `int3` instructions).
```
gdb> x/20i $rcx
0x555555555430 <int_arg>:	jmp    0x5555555552e0 <int_arg.cfi>
0x555555555435 <int_arg+5>:	int3
0x555555555436 <int_arg+6>:	int3
0x555555555437 <int_arg+7>:	int3
0x555555555438 <bad_int_arg>:	jmp    0x555555555310 <bad_int_arg.cfi>
0x55555555543d <bad_int_arg+5>:	int3
0x55555555543e <bad_int_arg+6>:	int3
0x55555555543f <bad_int_arg+7>:	int3
0x555555555440 <not_entry_point>:	jmp    0x5555555553a0 <not_entry_point.cfi>
0x555555555445 <not_entry_point+5>:	int3
0x555555555446 <not_entry_point+6>:	int3
0x555555555447 <not_entry_point+7>:	int3
```

The next step of instructions try to compute the index of the entry within the jumptable.
```
0x000000000000129a <+330>:	lea    rax,[rip+0x18f]        # 0x1430 <int_arg>
0x00000000000012a1 <+337>:	mov    rdx,rcx
0x00000000000012a4 <+340>:	sub    rdx,rax
0x00000000000012a7 <+343>:	mov    rax,rdx
0x00000000000012aa <+346>:	shr    rax,0x3
0x00000000000012ae <+350>:	shl    rdx,0x3d
0x00000000000012b2 <+354>:	or     rax,rdx
```

`rax` is initialized with the start address of the jumptable. The base address is
subtracted and divided by 8 to get the index. We can ignore the `shl` instruction
(and the subsequent `or`) for now.

Next, the index is (unsigned)compared against 2.

```
0x00000000000012b5 <+357>:	cmp    rax,0x2
0x00000000000012b9 <+361>:	jbe    0x12c0 <main+368>
0x00000000000012bb <+363>:	ud1    eax,DWORD PTR [eax+0x2]
0x00000000000012c0 <+368>:	mov    edi,DWORD PTR [rbp-0x8]
0x00000000000012c3 <+371>:	call   rcx
```
If the index is below or equal to 0x2, `call rcx` is executed.

## Using arguments `2`
This case simulates control flow hijacking to a function that has a different signature.

```bash
0x000000000000128f <+319>:	lea    rcx,[rip+0x2da2]        # 0x4038 <f>
0x0000000000001296 <+326>:	mov    rcx,QWORD PTR [rcx+rax*8]
```

The instructions load an address from the offset `2*8` from the start of `f`. Lets debug
and look at what rcx points to:
```
gdb$ x/10i $rcx
0x555555555350 <float_arg>:	push   rbp
0x555555555351 <float_arg+1>:	mov    rbp,rsp
0x555555555354 <float_arg+4>:	sub    rsp,0x10
0x555555555358 <float_arg+8>:	movss  DWORD PTR [rbp-0x4],xmm0
0x55555555535d <float_arg+13>:	lea    rdi,[rip+0xeaa]        # 0x55555555620e
0x555555555364 <float_arg+20>:	mov    al,0x0
0x555555555366 <float_arg+22>:	call   0x555555555030 <printf@plt>
0x55555555536b <float_arg+27>:	movss  xmm0,DWORD PTR [rbp-0x4]
0x555555555370 <float_arg+32>:	cvtss2sd xmm0,xmm0
0x555555555374 <float_arg+36>:	lea    rdi,[rip+0xeb8]        # 0x555555556233
```

Note that `rcx` does not point to the jumptable, but to the actual function itself. In the
subsequent instructions `sub rdx, rax`(i.e. `loaded_call_address - jump_table_base`) results
in a negative number, and results in an index value that does not make sense. The UD1
is executed, resulting in a SIGILL.

**Note to self**: Perhaps this indicates that a carelessly placed SIGILL handler can thwart
CFI. Might be a fun idea for a CTF challenge.

## Using arguments `3`
This case seems to be intended to simulate the case where control flow may be
redirected to a ROP gadget by overwriting the contents of a function pointer. It does not
seem that the example is able to demonstrate this when compiled with CFI.

Looking at the source we have:
```
    .not_entries = {(int_arg_fn)((uintptr_t)(not_entry_point)+0x20)}
```

When compiled with CFI, `not_entry_point` refers to the jumptable entry that calls
into `not_entry_point.cfi`(the actual function). To understand how CFI acts when a
function pointer is overwritten with a gadget address, we would need `.not_entries` to
be set to `not_entry_point.cfi + 0x20`.

```
=> 0x55555555528f <main+319>:	lea    rcx,[rip+0x2da2]        # 0x555555558038 <f>
   0x555555555296 <main+326>:	mov    rcx,QWORD PTR [rcx+rax*8]
```

In order to do this, I manually set `rcx` to `not_entry_point.cfi + 0x20` after the above
instructions are executed. As you might have guessed, the length check stops the `call`
and we get SIGILL instead.

## Final thoughts

We saw the following instructions above:
```
=> 0x5555555552ae <main+350>:	shl    rdx,0x3d
   0x5555555552b2 <main+354>:	or     rax,rdx
   0x5555555552b5 <main+357>:	cmp    rax,0x2
```

At this point `rdx` is the offset of the jumptable entry from the jumptable base. When
shift 61 bits to the left, the only bits remaining are LSB 3 bits(values range [0, 7]).
If any of these bits are set, the value in `rax` will end up being very large. Is this
the intent? I don't know.


