# Missed optimizations

This is a list of some missed optimizations in C compilers. To my knowledge,
these have not been reported before (but it is difficult to search for such
things, and people may not have bothered to write these up before). I have
reported some of these and/or developed more or less mature patches for
some; see links below.

These are *not* correctness bugs, only cases where a compiler might generate
simpler or otherwise more efficient code. None of these are likely to affect
the actual, measurable performance of real applications.

I have tested GCC, Clang, and CompCert, mostly targeting ARM at `-O3`.
Unless noted otherwise, the examples below can be reproduced with the
following versions and configurations:

Compiler | Version | Revision | Date | Configuration
---------|---------|----------|------|--------------
GCC      | 8.0.0 20170906 | r251801 | 2017-09-06 | `--target=armv7a-eabihf --with-arch=armv7-a --with-fpu=vfpv3-d16 --with-float-abi=hard --with-float=hard`
Clang    | 6.0.0 (trunk 312637) | LLVM r312635, Clang r312634 | 2017-09-06 | `--target=armv7a-eabihf -march=armv7-a -mfpu=vfpv3-d16 -mfloat-abi=hard`
CompCert | CompCert 3.0.1 | f3c60be | 2017-08-28 | `armv7a-linux`

For each example, at least one of these three (typically GCC or Clang)
produces the "right" code.


## GCC issues

###



## CompCert issues

### No dead code elimination for useless branches

```
void fn1(int r0) {
  if (r0 + 1)
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

The value of `v` is never used; this assignment is dead code. It
should not affect register allocation.

### Missed register coalescing

In the previous example, the copies for the arguments to the `mul` operation
(`r0` to `r1`, then on to `r2`) are redundant. They should be removed, and
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
the algebraic properties of operator's neutral and zero elements.

### Missing reassociation and folding of constants

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
