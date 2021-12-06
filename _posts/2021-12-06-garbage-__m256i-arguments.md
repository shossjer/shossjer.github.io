# Garbage `__m256i` arguments

Or, how to write a 1-bit random generator.

## Background

Writing multi-architecture support can be tricky. There are a lot of
details to get right and it does not help that all compilers deal with
it differently. In gcc and clang, for instance, you must explicitly
enable support for each _exotic_ instruction set you want to
use. There are many ways one can do this:

1. you can enable them globally with compiler options like `-mavx2`
   and `-march=haswell`,
2. enable them locally for a particular translating unit with pragmas
   like
   ```c++
   #pragma gcc target avx2
   ```
   or
3. decorate every function that needs to support explicitly like so
   ```c++
   __attribute__((target("avx2"))) int foo(int);
   ```

I tend to use the latter approach since that give me the best
flexibility and is the only way, _really_, if you write code in
headers. With all these different options, possible architectures,
mysterious instructions, and (I think) not that many people using
these features, it does not surprise me at all that things sometimes
does not work as expected.

## Problem

Consider the following two harmless code snippets, located in two
different translation units.

```c++
__attribute__((target("avx2")))
int is_zero(__m256i x)
{
    return _mm256_testz_si256(x, x); // returns 1 iff x is all zeros
}
```

```c++
__attribute__((target("avx2")))
int foo()
{
    extern __attribute__((target("avx2"))) int is_zero(__m256i x);

    const __m256i x = _mm256_setzero_si256(); // sets all bits to zero
    return is_zero(x);
}
```

What would you expect `foo()` to return? If you use gcc then you will
get a very reliable result of 1, hardly surprising. If instead you use
clang, you will either get a 1 or a 0!?

## Investigation

Let us look at some assembly and try to figure out what clang is
doing. The following assembly comes from compiling both snippets with
`-O2` and without any global options to target AVX2.

```asm
is_zero(long long __vector(4)):                        # @is_zero(long long __vector(4))
        push    rbp
        mov     rbp, rsp
        and     rsp, -32
        sub     rsp, 32
        vmovdqa ymm0, ymmword ptr [rbp + 16]
        xor     eax, eax
        vptest  ymm0, ymm0
        sete    al
        mov     rsp, rbp
        pop     rbp
        vzeroupper
        ret
foo():                                # @foo()
        push    rbp
        mov     rbp, rsp
        and     rsp, -32
        sub     rsp, 96
        vxorps  xmm0, xmm0, xmm0
        vmovaps ymmword ptr [rsp + 32], ymm0
        vmovaps ymm0, ymmword ptr [rsp + 32]
        vmovaps ymmword ptr [rsp], ymm0
        vzeroupper
        call    is_zero(long long __vector(4))
        mov     rsp, rbp
        pop     rbp
        ret
```

We see that `is_zero` expects the argument to be located on the stack
and that `foo` puts it there, kind of. `foo` seems to be doing a bunch
of unnecessary mucking around, and `is_zero` reserves stack space but
does not use it? What is going on!? This should be a very
straightforward piece of code. Why is the argument not simply passed
as is in the register `ymm0`? Well, guess what gcc is doing. Same
compiler options and gcc gives us this.

```asm
is_zero(long long __vector(4)):
        xor     eax, eax
        vptest  ymm0, ymm0
        sete    al
        ret
foo():
        vpxor   xmm0, xmm0, xmm0
        jmp     is_zero(long long __vector(4))
```

Yes! It is that easy: zero `xmm0` (the low 128-bits in `ymm0`), and
use `ymm0` *as is* in `is_zero`. No mucking about. How can we make
clang do this? The only way I have found is to globally enable
AVX2. When doing this clang generates identical code as gcc. And this
is where my problem began, early this morning when my program suddenly
started crashing randomly and the only change I had made since running
it previously was to pass an argument of type `__m256i` to a
function. You see, my version of `is_zero` was located in a
translation unit that was not compiled with a global option to enable
AVX2, but my version of `foo` was. So, `is_zero` tried to read data
from the stack that was not there and it caused much confusion down
the line of execution.

## Thoughts

To my knowledge, it is not possible to make clang generate the correct
and simple assembly that gcc does (without globally enable AVX2),
which is a huge problem. It makes writing support for
multi-architecture even more difficult than it already is! I hate it.

Since it is so rare for clang to be wrong about something, it made me
question whether I had made a mistake somewhere or that maybe gcc was
being overly relaxed about the situation. However, after spending a
day writing this up, I am almost certain that this is an oversight
from the clang devs, and a rather costly one at that! The overhead of
passing vector-types by value _on the stack_ is insane.

If they ever get their bug tracker working for dummies like me, I will
attempt to file a bug about this. Until then, be very careful when
mixing compiler options!

*Updated 2022-02-06*

A bug report was filed: https://github.com/llvm/llvm-project/issues/53619
