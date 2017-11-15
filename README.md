# Missed optimizations in C compilers

This is a list of some missed optimizations in C compilers. To my knowledge,
these have not been reported by others (but it is difficult to search for
such things, and people may not have bothered to write these up before). I
have reported some of these in bug trackers and/or developed more or less
mature patches for some; see links below.

These are *not* correctness bugs, only cases where a compiler misses an
opportunity to generate simpler or otherwise (seemingly) more efficient
code. None of these are likely to affect the actual, measurable performance
of real applications. Also, I have not measured most of these: The seemingly
inefficient code may somehow turn out to be faster on real hardware.

I have tested GCC, Clang, and CompCert, mostly targeting ARM at `-O3`. The
exact versions tested are specified below. For each example, at least one of
these three compilers (typically GCC or Clang) produces the "right" code.

This list was put together by Gergö Barany <gergo.barany@inria.fr>. I'm
interested in feedback.
I have described how I found these missed optimizations in a [preprint
(PDF)](missed_optimizations_preprint.pdf). The software described in the
paper is not released yet. I also made a quick-and-dirty [poster
(PDF)](missed_optimizations_poster.pdf).

Since initial publication, there have been some insightful comments on this
list [on Reddit](https://www.reddit.com/r/programming/comments/6ylrpi/missed_optimizations_in_c_compilers/)
and [on Hacker News](https://news.ycombinator.com/item?id=15187505).


# Added 2017-11-04

Unless noted otherwise, the examples below can be reproduced with the
following compiler versions and configurations:

Compiler | Version | Revision | Date | Configuration
---------|---------|----------|------|--------------
GCC      | 8.0.0 20171101 | r254306 | 2017-11-01 | `--target=armv7a-eabihf --with-arch=armv7-a --with-fpu=vfpv3-d16 --with-float-abi=hard --with-float=hard`
Clang    | 6.0.0 (trunk 317088) | LLVM r317088, Clang r317076 | 2017-11-01 | `--target=armv7a-eabihf -march=armv7-a -mfpu=vfpv3-d16 -mfloat-abi=hard`
CompCert | 3.1 | 0c00ace | 2017-10-26 | `armv7a-linux`

## GCC

### Two dead stores for type conversion

Similar to a case below, but maybe more striking:
```
unsigned fn1(float p1, double p2) {
  unsigned a = p2;
  if (p1)
    a = 8;
  return a;
}
```

GCC predicates the entire function, nicely moving the conversion of `p2` to
`unsigned` into a branch. But for some reason, in both branches it spills
the value for `a` to the stack, without ever reloading it:
```
    vcmp.f32    s0, #0
    sub sp, sp, #8
    vmrs    APSR_nzcv, FPSCR
    vcvteq.u32.f64  s15, d1
    movne   r3, #8
    strne   r3, [sp, #4]
    movne   r0, r3
    vmoveq  r0, s15 @ int
    vstreq.32   s15, [sp, #4]   @ int
    add sp, sp, #8
    @ sp needed
    bx  lr
```

### Missed simplification of modulo-and

```
short fn1(int p1) {
  signed a = (p1 % 4) & 1;
  return a;
}
```

GCC computes modulo 4 as bitwise-and 3, carefully handling the case where
`p1` might be negative, then does the bitwise-and 1:
```
    rsbs    r3, r0, #0
    and r0, r0, #3
    and r3, r3, #3
    rsbpl   r0, r3, #0
    and r0, r0, #1
```

But the result of `(p1 % 4) & 1` is 1 iff `p1 % 4` is odd iff `p1` is odd
iff `p1 & 1` is 1, so Clang just computes this directly:

```
    and r0, r0, #1
```

### Missed simplification of division in condition

In the function:
```
char fn1(char p1) {
  signed a = 4;
  if (p1 / (p1 + 3))
    a = 1;
  return a;
}
```
GCC misses the fact that the condition is always false because `p1 + 3` is
always greater than `p1` (as `char` is unsigned on ARM), and generates code
to evaluate it:
```
    add r1, r0, #3
    bl  __aeabi_idiv
    cmp r0, #0
    moveq   r0, #4
    movne   r0, #1
```

Clang returns 4 unconditionally.

### Missed simplification/floating-point constant propagation?

```
float fn1(short p1) {
  char a;
  float b, c;
  a = p1;
  b = c = 765;
  if (a == b)
    c = 0;
  return c;
}
```

At the comparison `a == b`, `a` is a `char`, and it cannot hold a value that
compares equal to `b`'s value of 765.

GCC computes the comparison anyway, Clang returns 765.0f unconditionally.

### Missed bitwise tricks

```
char fn1(short p1, int p2) {
  long a;
  int v;
  a = p2 & (p1 | 415615);
  v = a << 1;
  return v;
}
```

The least significant byte of 415615 is `0x7f`, so the lowest 7 bits of `a`
are the same as the lowest 7 bits of `p2`, hence the lowest 8 bits of `v`
are the same as the lowest 8 bits of `p2 << 1`, and that is all that is
needed for the result. GCC generates all the operations present at the
source level, while Clang simplifies nicely:
```
    lsl r0, r1, #1
    uxtb    r0, r0
```

### Unnecessary spilling due to badly scheduled move-immediates

This is just a much smaller example of an issue with the same title below.

```
long fn1(char p1, short p2, long p3, long p4, short p5, long p6) {
  long a, b = 1000893;
  a = 600 - p2 * 3;
  if (p4 >= p5 * !(p6 * a))
    b = p3;
  return b;
}
```

GCC loads the constant 1000893 into a register early on, causing it to spill
a register, but only uses the value much later and conditionally:

```
    str lr, [sp, #-4]!      @ spill lr
    movw    lr, #17853      @ move lower part of 1000893 into lr
    ldr r1, [sp, #8]
    movt    lr, 15          @ move upper part of 1000893 into lr
    ...
    cmp r1, r3
    movle   r0, r2
    movgt   r0, lr          @ use 1000893
```

Clang just loads the constant conditionally, at the point where it is
actually needed, and avoids spilling:

```
    ...
    movwgt  r2, #17853
    movtgt  r2, #15
    mov r0, r2
```


## Clang

### Missed simplification of integer-valued floating-point arithmetic

In some cases (see one further below), Clang can perform floating-point
arithmetic in integers where the result is an exact integer; GCC doesn't
usually do this. But here is a counterexample:

```
char fn1(char p1) {
  float a;
  short b;
  a = !p1;
  b = -a;
  return b;
}
```

Clang converts to `float`, negates, and converts back:
```
    vmov.f32    s2, #1.000000e+00
    vldr    s0, .LCPI0_0        @ load 0.0
    cmp r0, #0
    vmoveq.f32  s0, s2
    vneg.f32    s0, s0
    vcvt.s32.f32    s0, s0
    vmov    r0, s0
```

GCC doesn't:
```
    cmp r0, #0
    moveq   r0, #255
    movne   r0, #0
```

### Missed conditional constant propagation on `double`

```
short fn1(double p1) {
  unsigned a = 4;
  if (p1 == 604884)
    if (0 * p1)
      a = 7;
  return a;
}
```

If `p1` is 604884, then `0 * p1` is false. Clang evaluates both conditions,
GCC returns 4 unconditionally.


### Missed simplification of bitwise operations

```
float fn1(short p1) {
  float a = 516.403544893f;
  if (p1 & 6 >> (p1 & 31) ^ ~0)
    a = 1;
  return a;
}
```

Clang evaluates the branch condition because it misses that `((p1 & (6 >>
(p1 & 31))) ^ ~0)` is nonzero because almost all the bits of the inner
expression are zero.


## CompCert

### Reluctance to select `mvn` instruction

In this function:
```
int fn1(int p1) {
  return -7 - p1;
}
```
CompCert computes the subtraction by adding a sequence of large constants:
```
    rsb r0, r0, #249
    add r0, r0, #65280
    add r0, r0, #16711680
    add r0, r0, #-16777216
```

It's simpler to synthesize the constant -7 as the negation of 6 and
subtracting, as GCC does:
```
    mvn r3, #6
    sub r0, r3, r0
```


# Added 2017-09-06

Unless noted otherwise, the examples below can be reproduced with the
following compiler versions and configurations:

Compiler | Version | Revision | Date | Configuration
---------|---------|----------|------|--------------
GCC      | 8.0.0 20170906 | r251801 | 2017-09-06 | `--target=armv7a-eabihf --with-arch=armv7-a --with-fpu=vfpv3-d16 --with-float-abi=hard --with-float=hard`
Clang    | 6.0.0 (trunk 312637) | LLVM r312635, Clang r312634 | 2017-09-06 | `--target=armv7a-eabihf -march=armv7-a -mfpu=vfpv3-d16 -mfloat-abi=hard`
CompCert | 3.0.1 | f3c60be | 2017-08-28 | `armv7a-linux`

## GCC

### Useless initialization of struct passed by value

```
struct S0 {
  int f0;
  int f1;
  int f2;
  int f3;
};

int f1(struct S0 p) {
    return p.f0;
}
```

The struct is passed in registers, and the function's result is already in
`r0`, which is also the return register. The function could return
immediately, but GCC first stores all the struct fields to the stack and
reloads the first field:

```
    sub sp, sp, #16
    add ip, sp, #16
    stmdb   ip, {r0, r1, r2, r3}
    ldr r0, [sp]
    add sp, sp, #16
    bx  lr
```

Reported at https://gcc.gnu.org/bugzilla/show_bug.cgi?id=80905.

### `float` to `char` type conversion goes through memory

```
char fn1(float p1) {
  return (char) p1;
}
```
results in:
```
    vcvt.u32.f32    s15, s0
    sub sp, sp, #8
    vstr.32 s15, [sp, #4]   @ int
    ldrb    r0, [sp, #4]    @ zero_extendqisi2
```
i.e., the result of the conversion in `s15` is stored to the stack and then
reloaded into `r0` instead of just copying it between the registers (and
possibly truncating to `char`'s range). Same for `double` instead of
`float`, but *not* for `short` instead of `char`.

Reported at https://gcc.gnu.org/bugzilla/show_bug.cgi?id=80861.

If you are worried about the conversion overflowing, you might prefer this
version:

```
#include <math.h>
#include <limits.h>

char fn1(float p1) {
  if (isfinite(p1) && CHAR_MIN <= p1 && p1 <= CHAR_MAX) {
    return (char) p1;
  }
  return 0;
}
```

GCC still goes through memory to perform the zero extension.


### Spill instead of register copy

```
int N;
int fn2(int p1, int p2) {
  int a = p2;
  if (N)
    a *= 10.5;
  return a;
}
```

GCC spills `a` although enough integer registers would be available:
```
    str r1, [sp, #4]    // initialize a to p2
    cmp r3, #0
    beq .L13
    ... compute multiplication, put into d7 ...
    vcvt.s32.f64    s15, d7
    vstr.32 s15, [sp, #4]   @ int   // update a in branch
.L13:
    ldr r0, [sp, #4]    // reload a
```

Reported at https://gcc.gnu.org/bugzilla/show_bug.cgi?id=81012 (together
with another issue further below).

### Missed simplification of multiplication by integer-valued floating-point constant

Variant of the above code with the constant changed slightly:
```
int N;
int fn5(int p1, int p2) {
  int a = p2;
  if (N)
    a *= 10.0;
  return a;
}
```

GCC converts `a` to double and back as above, but the result must be the
same as simply multiplying by the integer 10. Clang realizes this and
generates an integer multiply, removing all floating-point operations.

If you are worried about the multiplication overflowing, you might prefer
this version:

```
#include <limits.h>

int N;
int fn5(int p1, int p2) {
  int a = p2;
  if (N && INT_MIN / 10.0 <= a && a <= INT_MAX / 10.0)
    a *= 10.0;
  return a;
}
```

Clang still generates code for multiplying integers, while GCC multiplies as
`double`.

### Dead store for `int` to `double` conversion

```
double fn3(int p1, short p2) {
  double a = p1;
  if (1881 * p2 + 5)
    a = 2;
  return a;
}
```

`a` is initialized to the value of `p1`, and GCC spills this value. The
spill store is dead, the value is never reloaded:

```
    ...
    str r0, [sp, #4]
    cmn r1, #5
    vmovne.f64  d0, #2.0e+0
    vmoveq  s15, r0 @ int
    vcvteq.f64.s32  d0, s15
    add sp, sp, #8
    @ sp needed
    bx  lr
```

Reported at https://gcc.gnu.org/bugzilla/show_bug.cgi?id=81012 (together
with another issue above).

### Missed strength reduction of division to comparison

```
unsigned int fn1(unsigned int a, unsigned int p2) {
  int b;
  b = 3;
  if (p2 / a)
    b = 7;
  return b;
}
```

The condition `p2 / a` is true (nonzero) iff `p2 >= a`. Clang compares, GCC
generates a division (on ARM, x86, and PowerPC).

### Missed simplification of floating-point addition

```
long fn1(int p1) {
  double a;
  long b = a = p1;
  if (0 + a + 0)
    b = 9;
  return b;
}
```

It is not correct in general to replace `a + 0` by `a` in floating-point
arithmetic due to NaNs and negative zeros and whatnot, but here `a` is
initialized from an `int`'s value, so none of these cases can happen. GCC
generates all the floating-point operations, while Clang just compares `p1`
to zero.

### Missed reassociation of multiplications

```
double fn1(double p1) {
  double v;
  v = p1 * p1 * (-p1 * p1);
  return v;
}

int fn2(int p1) {
  int v;
  v = p1 * p1 * (-p1 * p1);
  return v;
}
```

For each function, GCC generates three multiplications:

```
    vnmul.f64   d7, d0, d0
    vmul.f64    d0, d0, d0
    vmul.f64    d0, d7, d0
```
and
```
    rsb r2, r0, #0
    mul r3, r0, r0
    mul r0, r0, r2
    mul r0, r3, r0
```

Clang can do the same with two multiplications each:

```
    vmul.f64    d0, d0, d0
    vnmul.f64   d0, d0, d0
```
and
```
    mul r0, r0, r0
    rsb r1, r0, #0
    mul r0, r0, r1
```

### Heroic loop optimization instead of removing the loop

```
int N;
double fn1(double *p1) {
  int i = 0;
  double v = 0;
  while (i < N) {
    v = p1[i];
    i++;
  }
  return v;
}
```

This function returns 0 if `N` is 0 or negative; otherwise, it returns
`p1[N-1]`. On x86-64, GCC generates a complex unrolled loop (and a simpler
loop on ARM). Clang removes the loop and replaces it by a simple branch.
GCC's code is too long and boring to show here, but it's similar enough to
https://godbolt.org/g/RYwgq4.

*Note:* An earlier version of this document claimed that GCC also removed the
loop on ARM. It was pointed out to me that this claim was false, GCC does
generate a loop on ARM too. My (stupid) mistake, I had misread the assembly
code.

### Unnecessary spilling due to badly scheduled move-immediates

```
double fn1(double p1, float p2, double p3, float p4, double *p5, double p6)
{
  double a, c, d, e;
  float b;
  a = p3 / p1 / 38 / 2.56380695236e38f * 5;
  c = 946293016.021f / a + 9;
  b = p1 - 6023397303.74f / p3 * 909 * *p5;
  d = c / 683596028.301 + p3 - b + 701.296936339 - p4 * p6;
  e = p2 + d;
  return e;
}
```

Sorry about the mess. GCC generates a bunch of code for this, including:

```
    vstr.64 d10, [sp]
    vmov.f64    d11, #5.0e+0
    vmov.f64    d13, #9.0e+0
    ... computations involving d10 but not d11 or d13 ...
    vldr.64 d10, [sp]
    vcvt.f64.f32    d6, s0
    vmul.f64    d11, d7, d11
    vdiv.f64    d5, d12, d11
    vadd.f64    d13, d5, d13
    vdiv.f64    d0, d13, d14
```

The register `d10` must be spilled to make room for the constants 5 and 9
being loaded into `d11` and `d13`, respectively. But these loads are much
too early: Their values are only needed after `d10` is restored. These move
instructions (at least one of them) should be closer to their uses, freeing
up registers and avoiding the spill.

### Missed constant propagation (or so it seemed)

```
int fn1(int p1) {
  int a = 6;
  int b = (p1 / 12 == a);
  return b;
}
int fn2(int p1) {
  int b = (p1 / 12 == 6);
  return b;
}
```

These functions should be compiled identically. The condition is equivalent
to `6 * 12 <= p1 < 7 * 12`, and the two signed comparisons can be replaced
by a subtraction and a single unsigned comparison:

```
    sub r0, r0, #72
    cmp r0, #11
    movhi   r0, #0
    movls   r0, #1
    bx  lr
```

GCC used to do this for the first function, but not the second one. It
seemed like constant propagation failed to replace `a` by 6.

Reported at https://gcc.gnu.org/bugzilla/show_bug.cgi?id=81346 and fixed in
revision r250342; it turns out that this pattern used to be implemented
(only) in a very early folding pass that ran before constant propagation.

### Missed tail-call optimization for compiler-generated call to intrinsic

```
int fn1(int a, int b) {
  return a / b;
}
```

On ARM, integer division is turned into a function call:

```
    push    {r4, lr}
    bl  __aeabi_idiv
    pop {r4, pc}
```

In the code above, GCC allocates a stack frame, not realizing that this call
in tail position could be simply
```
    b   __aeabi_idiv
```

(When the call, even to `__aeabi_idiv`, appears in the source code, GCC is
perfectly capable of compiling it as a tail call.)


## Clang

### Incomplete range analysis for certain types

```
short fn1(short p) {
  short c = p / 32769;
  return c;
}

int fn2(signed char p) {
  int c = p / 129;
  return c;
}
```

In both cases, the divisor is outside the possible range of values for `p`,
so the function's result is always 0. Clang doesn't realize this and
generates code to compute the result. (It does optimize corresponding cases
for unsigned `short` and `char`.)

A report with this and all the other range analysis examples below was filed
by [@gnzlbg](https://github.com/gnzlbg) at
https://bugs.llvm.org/show_bug.cgi?id=34517, and most cases have been fixed.

### More incomplete range analysis

```
int fn1(short p4, int p5) {
  int v = 0;
  if ((p4 - 32779) - !p5)
    v = 7;
  return v;
}
```

The condition is always true, and the function always returns 7. Clang
generates code to evaluate the condition. Similar to the case above, but
seems to be more complex: If the `- !p5` is removed, Clang also realizes
that the condition is always true.

### Even more incomplete range analysis

```
int fn1(int p, int q) {
  int a = q + (p % 6) / 9;
  return a;
}

int fn2(int p) {
  return (1 / p) / 100;
}
int fn3(int p5, int p6) {
  int a, c, v;
  v = -!p5;
  a = v + 7;
  c = (4 % a);
  return c;
}
int fn4(int p4, int p5) {
  int a = 5;
  if ((5 % p5) / 9)
    a = p4;
  return a;
}
```

The first function always returns `q`, the others always return 0, 4, and 5,
respectively. Clang evaluates at least one division or modulo operation in
each case.

(I have a *lot* more examples like this, but this list is boring enough as
it is.)

### Redundant code generated for `double` to `int` conversion

```
int fn1(double c, int *p, int *q, int *r, int *s) {
  int i = (int) c;
  *p = i;
  *q = i;
  *r = i;
  *s = i;
  return i;
}
```

There is a single type conversion on the source level. Clang duplicates the
conversion operation (`vcvt`) for each use of `i`:

```
    vcvt.s32.f64    s2, d0
    vstr    s2, [r0]
    vcvt.s32.f64    s2, d0
    vstr    s2, [r1]
    vcvt.s32.f64    s2, d0
    vstr    s2, [r2]
    vcvt.s32.f64    s2, d0
    vcvt.s32.f64    s0, d0
    vmov    r0, s0
    vstr    s2, [r3]
```

Several times, the result of the conversion is written into the register
that already holds that same value (`s2`).

Reported at https://bugs.llvm.org/show_bug.cgi?id=33199, fixed.

### Missing instruction selection pattern for `vnmla`

```
double fn1(double p1, double p2, double p3) {
  double a = -p1 - p2 * p3;
  return a;
}
```

Clang generates
```
    vneg.f64    d0, d0
    vmls.f64    d0, d1, d2
```
but this could just be
```
    vnmla.f64   d0, d1, d2
```

The instruction selector has a pattern for selecting `vnmla` for the
expression `-(a * b) - dst`, but not for the symmetric `-dst - (a * b)`.

Patch submitted: https://reviews.llvm.org/D35911


## CompCert

### No dead code elimination for useless branches

```
void fn1(int r0) {
  if (r0 + 42)
    0;
}
```

The conditional branch due to the `if` statement is removed, but the
computation for its condition remains in the generated code:

```
    add r0, r0, #42
```

This appears to be because dead code elimination runs before useless
branches are identified and eliminated.

Fixed in
https://github.com/AbsInt/CompCert/commit/93d2fc9e3b23d69c0c97d229f9fa4ab36079f507
due to my report.

### Missing constant folding patterns for modulo

```
int fn4(int p1) {
  int a = 0 % p1;
  return a;
}

int fn5(int a) {
  int b = a % a;
  return b; 
} 
```

In both cases, CompCert generates a modulo operation (on ARM, it generates a
library call, a multiply, and a subtraction). In both cases, this operation
could be constant folded to zero.

My patch at 
https://github.com/gergo-/CompCert/commit/78555eb7afacac184d94db43c9d4438d20502be8
fixes the first one, but not the second. It also fails to deal with the
equivalent case of
```
  int zero = 0;
  int a = zero % p1;
```
because, by the time the constant propagator runs, the modulo operation has
been mangled into a complex call/multiply/subtract sequence that is
inconvenient to recognize.

Status: CompCert developers are (understandably) uninterested in this
trivial issue.

### Failure to generate `movw` instruction

```
int fn1(int p1) {
  int a = 506 * p1;
  return a;
}
```
results in:
```
    mov r1, #250
    orr r1, r1, #256
    mul r0, r0, r1
```

Two instructions are spent on loading the constant. However, ARM's `movw`
instruction can take an immediate between 0 and 65535, so this can be
simplified to:

```
    movw r1, #506
```

Fixed in a series of patches from me, merged at
https://github.com/AbsInt/CompCert/commit/c4dcf7c08016f175ba6c06d20c530ebaaad67749.

### Failure to constant fold `mla`

Call this code `mla.c`:

```
int fn1(int p1) {
  int a, b, c, d, e, v, f;
  a = 0;
  b = c = 0;
  d = e = p1;
  v = 4;
  f = e * d | a * p1 + b;
  return f;
}
```

The expression `a * p1 + b` evaluates to `0 * p1 + 0`, i.e., 0. CompCert is
able to fold both `0 * x` and `x + 0` in isolation, but on ARM it generates
an `mla` (multiply-add) instruction, for which it has no constant folding
rules:

```
    mov r4, #0
    mov r12, #0
    ...
    mla r2, r4, r0, r12
```

Fixed in a patch from me, merged at
https://github.com/AbsInt/CompCert/commit/4f46e57884a909d2f62fa7cea58b3d933a6a5e58.

### Unnecessary spilling due to dead code

On `mla.c` from above without the `mla` constant folding patch, CompCert
spills the `r4` register which it then uses to store a temporary:

```
    str r4, [sp, #8]
    mov r4, #0
    mov r12, #0
    mov r1, r0
    mov r2, r1
    mul r3, r2, r1
    mla r2, r4, r0, r12
    orr r0, r3, r2
    ldr r4, [sp, #8]
```

However, the spill goes away if the line `v = 4;` is removed from the
program:

```
    mov r3, #0
    mov r12, #0
    mov r1, r0
    mov r2, r1
    mul r1, r1, r2
    mla r12, r3, r0, r12
    orr r0, r1, r12
```

The value of `v` is never used. This assignment is dead code, and it is
compiled away. It should not affect register allocation.

### Missed register coalescing

In the previous example, the copies for the arguments to the `mul` operation
(`r0` to `r1`, then on to `r2`) are redundant. They could be removed, and
the multiplication written as just `mul r1, r0, r0`.

### Failure to propagate folded constants

Continuing the `mla.c` example further, but this time after applying the
`mla` constant folding patch, CompCert generates:

```
    mov r1, r0
    mul r2, r1, r0
    mov r1, #0
    orr r0, r2, r1
```

The `x | 0` operation is redundant. CompCert is able to fold this
operation if it appears in the source code, but in this case the 0 comes
from the previously folded `mla` operation. Such constants are not
propagated by the "constant propagation" (in reality, only local folding)
pass. Values are only propagated by the value analysis that runs before, but
for technical reasons the value analysis cannot currently take advantage of
the algebraic properties of operators' neutral and zero elements.

### Missed reassociation and folding of constants

```
int fn1(int p1) {
  int a, v, b;
  v = 1;
  a = 3 * p1 * 2 * 3;
  b = v + a * 83;
  return b;
}
```

All the multiplications could be folded into a single multiplication of `p1`
by 1494, but CompCert generates a series of adds and shifts before a
multiplication by 83:

```
    add r3, r0, r0, lsl #1
    mov r0, r3, lsl #1
    add r1, r0, r0, lsl #1
    mov r2, #83
    mla r0, r2, r1, r12
```
