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

I have tested GCC, Clang, and CompCert, mostly targeting ARM at `-O3`.
Unless noted otherwise, the examples below can be reproduced with the
following versions and configurations:

Compiler | Version | Revision | Date | Configuration
---------|---------|----------|------|--------------
GCC      | 8.0.0 20170906 | r251801 | 2017-09-06 | `--target=armv7a-eabihf --with-arch=armv7-a --with-fpu=vfpv3-d16 --with-float-abi=hard --with-float=hard`
Clang    | 6.0.0 (trunk 312637) | LLVM r312635, Clang r312634 | 2017-09-06 | `--target=armv7a-eabihf -march=armv7-a -mfpu=vfpv3-d16 -mfloat-abi=hard`
CompCert | CompCert 3.0.1 | f3c60be | 2017-08-28 | `armv7a-linux`

For each example, at least one of these three compilers (typically GCC or
Clang) produces the "right" code.

This list was put together by Gergö Barany <gergo.barany@inria.fr>. I'm
interested in feedback.


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
`p1[N-1]`. GCC compiles it as such on ARM (with no loop), but on x86-64, it
generates a complex unrolled loop. The code is too long and boring to show
here, but it's similar enough to https://godbolt.org/g/RYwgq4.

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

Reported at https://bugs.llvm.org/show_bug.cgi?id=33199.

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

I have a patch at
https://github.com/gergo-/CompCert/commit/8aa7b1ce3b0228232e027773752727f61b42a847.

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

I have a patch at
https://github.com/gergo-/CompCert/commit/37e7fd10fdb737dc4b5c07ee2f14aeffa51a6d75.

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
