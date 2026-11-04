# Bit Twiddling Hacks, revisited

This is Sean Eron Anderson's [Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html)
collection, with every hack in one place, the bugs I found in it fixed, and
six hacks swapped for cheaper versions.

I ported all the hacks to Rust to benchmark them. Along the way I found 14
bugs in the original C. A few give wrong answers, one reads past the end of
an array, several are undefined behaviour that compilers are free to break,
and a handful only fail on some platforms. Separately, six hacks turned out to
have versions that are better on every measure I tried. The short lists are
below, followed by every hack in full.

The code snippets on the original page are public domain. The explanations
here are my own.

## What changed

### Bugs

| # | Hack | Problem |
|---|---|---|
| 1 | [Select the bit with a given rank](#select-position-of-the-bit-with-a-given-rank) | The `if` version the page suggests returns wrong positions. The first count isn't masked to 8 bits. |
| 2 | [Modulus by (1 << s) - 1 in parallel](#by-1--s---1-in-parallel) | The alternative ending `m & -((signed)(m - d) >> s)` returns `m & 1`. The minus sign shouldn't be there. |
| 3 | [Modulus by (1 << s) - 1 in parallel](#by-1--s---1-in-parallel) | With s = 1 the loop runs past the end of the `Q` and `R` tables, which AddressSanitizer reports as a global buffer overflow. |
| 4 | [Sign extension in 3 operations](#from-a-variable-bit-width-in-3-operations) | `x * multipliers[b]` overflows `int` for every negative value. gcc and clang turn `(x * m) / m` into `x`, so -1 comes out as 31. |
| 5 | [Sign extension, shift version](#from-a-variable-bit-width) | `(x << m) >> m` shifts into the sign bit, and `(x << -b) >> -b` uses a negative shift count. The original calls them non-portable, but both are undefined behaviour. |
| 6 | [abs without branching](#absolute-value-without-branching) | Both versions overflow for `INT_MIN`, which is the case the unsigned result was meant to handle. |
| 7 | [Reverse bits by lookup table](#by-lookup-table) | `table[...] << 24` overflows `int` for every odd input. |
| 8 | [Interleave bits by table lookup](#by-table-lookup) | `table[y >> 8] << 17` overflows `int` for y >= 0x8000. |
| 9 | [Modulus by (1 << s) - 1](#by-1--s---1-without-division) | `(1 << s) - 1` overflows for s = 31. The page already fixed the same thing elsewhere with `1U`. |
| 10 | [Count trailing zeros in parallel](#in-parallel-3) | `-signed(v)` is C++ syntax. In C++ it overflows for v = 0x80000000. |
| 11 | [Rank](#rank-bits-set-from-the-msb-up-to-a-position) and [select](#select-position-of-the-bit-with-a-given-rank) | The masks are built from `~0UL`, which is 32 bits on Windows and 32-bit targets. `rank(~0, 64)` returns 0 there. |
| 12 | [Has a byte equal to n](#has-a-byte-equal-to-n-less-than-n-greater-than-n-or-between-m-and-n) | `hasvalue` uses `~0UL/255` but calls a `haszero` with fixed 32-bit constants. On 64-bit Linux and macOS it only checks four of the eight bytes. |
| 13 | [log2 with a lookup table](#with-a-lookup-table) | The table is plain `char`. On ARM and PowerPC Linux, `char` is unsigned, so log2(0) returns 255 instead of -1. |
| 14 | [log2 of a float](#of-a-32-bit-ieee-float) | The comment promises `int(log2(v))`, which truncates toward zero. The code returns the floor. For v = 0.3 that's -2 against -1. |

Each bug was reproduced on the original code as published, copied verbatim
into small test programs. They were built with gcc 15.2 and clang 21.1 on
x86-64 Linux, with MinGW for the Windows case and `-funsigned-char` for the
ARM case. Undefined behaviour was confirmed with UBSan or ASan, not by
reading the code. Every fix below was checked as well, exhaustively over all
2^32 inputs where that's possible. For bug 3 that meant confirming that
s = 2 to 31 never needs more than the six table entries, across all
128,849,018,880 combinations.

### Faster versions

The page counts every C operator as one operation, and it says to benchmark
anything that matters. I only kept a replacement if it beat the original on
all three: fewer operations by the page's count, fewer x86-64 instructions as
compiled, and faster in two separate benchmark runs whose confidence
intervals didn't overlap.

| Hack | Operations | Instructions | Time per call |
|---|---|---|---|
| [Is v a power of 2](#is-v-a-power-of-2) | 4 to 3 | 6 to 4 | 13% and 20% faster |
| [Count bits in a 32-bit word with 64-bit multiply/modulus](#with-64-bit-multiply-and-modulus) | 15 to 10 | 34 to 18 | about 50% faster |
| [Parity of a 32-bit word with a multiply](#parity-of-a-word-with-a-multiply) | 8 to 7 | 10 to 7 | 6-7% faster |
| [Parity of a 64-bit word with a multiply](#parity-of-a-word-with-a-multiply) | 8 to 7 | 11 to 9 | 5-6% faster |
| [log2 using a 64-bit float](#with-a-64-bit-ieee-float) | 5 to 4 | 8 to 5 | 23-24% faster |
| [log10](#log-base-10) | 9, with no extra tables | 42 to 10 | 15-30% faster |

The timings and instruction counts come from Rust ports of the same code,
benchmarked with criterion on an AMD Ryzen 7 5825U (rustc 1.99,
`-C target-cpu=native`). The ports keep every C operation as written; I
checked the assembly to make sure the compiler hadn't swapped any of them
for a single built-in instruction.
Correctness was checked exhaustively over all 2^32 inputs, or with the z3
solver for the 64-bit parity.

Plenty of candidates had fewer operations but failed the other two tests.
Usually a `%` by a constant was the culprit: it counts as one operation but
compiles to five or so instructions. Or a shorter version just had a longer
dependency chain. HAKMEM's popcount, for example, uses 10 operations against
the "best" method's 12, but it is slower, so it only replaces the page's
mul/mod method above. Knuth's 21-operation bit reversal loses to the
23-operation parallel version. A `double` version of round-up-to-power-of-2
saves one operation and three instructions, but it isn't measurably faster,
so it's left out.

### How everything was checked

Every C block in this file was extracted and compiled as one program (gcc
and clang, `-Wall -Wextra`, UBSan and ASan). About 58 million checks then
compared each function against a simple reference implementation, on edge
cases and random inputs. A coverage check made sure no function was missed,
and some deliberately broken copies of the file confirmed the tests catch
mistakes.

## Conventions

`int` and `unsigned int` are 32 bits, `CHAR_BIT` is 8, and `long long` is 64
bits. A few hacks rely on `>>` of a negative `int` being an arithmetic shift.
C leaves that to the implementation, and the original says so where it
matters; I've kept those notes. Reading a float's bits uses `memcpy`, which
is the portable form the original recommends over pointer casts.

On counting operations: each C operator counts as one, including comparisons,
`!`, `&&`, `?:`, unary minus, `%`, `/` and array indexing. Converting between
int and float counts as one, and so does reinterpreting the bits. Casts between
signed and unsigned of the same width don't count, and nor does widening.
Assigning an intermediate value is free. It's a rough model, as the original
admits. A division costs far more than an addition.

## Contents

- [Sign and comparison](#sign-and-comparison)
- [Sign extension](#sign-extension)
- [Conditional set, negate and merge](#conditional-set-negate-and-merge)
- [Counting bits set](#counting-bits-set)
- [Parity](#parity)
- [Swapping](#swapping)
- [Reversing bits](#reversing-bits)
- [Modulus division](#modulus-division)
- [Log base 2](#log-base-2)
- [Log base 10](#log-base-10)
- [Log base 2 of a float](#log-base-2-of-a-float)
- [Counting trailing zeros](#counting-trailing-zeros)
- [Rounding up to a power of 2](#rounding-up-to-a-power-of-2)
- [Interleaving bits](#interleaving-bits)
- [Bytes in a word](#bytes-in-a-word)
- [Next bit permutation](#next-bit-permutation)

```c
#include <limits.h>
#include <stdbool.h>
#include <stdint.h>
#include <string.h>
```

## Sign and comparison

### Sign of an integer

`-(v < 0)` is the obvious way. The shift versions avoid the comparison, at
the cost of relying on arithmetic `>>`. The unsigned-shift form doesn't need
that, and it avoids a branch on CPUs with flag registers.

```c
int sign_obvious(int v)        { return -(v < 0); }   /* -1 if negative, else 0 */
/* Avoids a branch on CPUs with flag registers (IA-32): */
int sign_unsigned_shift(int v) { return -(int)((unsigned int)((int)v) >> (sizeof(int) * CHAR_BIT - 1)); }
/* One operation fewer, but relies on arithmetic >>: */
int sign_shift(int v)          { return v >> (sizeof(int) * CHAR_BIT - 1); }
/* -1 or +1: */
int sign_plus_minus(int v)     { return +1 | (v >> (sizeof(int) * CHAR_BIT - 1)); }
/* -1, 0 or +1: */
int sign3_portable(int v)      { return (v != 0) | -(int)((unsigned int)((int)v) >> (sizeof(int) * CHAR_BIT - 1)); }
int sign3_shift(int v)         { return (v != 0) | (v >> (sizeof(int) * CHAR_BIT - 1)); }
int sign3(int v)               { return (v > 0) - (v < 0); }
/* +1 if non-negative, else 0: */
int is_non_negative(int v)     { return 1 ^ ((unsigned int)v >> (sizeof(int) * CHAR_BIT - 1)); }
```

### Opposite signs

```c
bool opposite_signs(int x, int y) { return (x ^ y) < 0; }
```

### Absolute value without branching

`mask` is all ones when v is negative and zero otherwise, so adding it and
XORing with it negates v only when needed.

Fixed (bug 6): the original computes `v + mask` in `int`, which overflows for
`INT_MIN`. Doing the arithmetic in `unsigned int` gives 2^31, as intended.

```c
unsigned int abs_branchless(int v) {
  int const mask = v >> (sizeof(int) * CHAR_BIT - 1);
  return ((unsigned int)v + mask) ^ mask;
}
/* Patented variation (US 6,073,150; likely invalid, see the original's notes): */
unsigned int abs_branchless_alt(int v) {
  int const mask = v >> (sizeof(int) * CHAR_BIT - 1);
  return ((unsigned int)v ^ mask) - mask;
}
/* The obvious approach, for comparison: */
unsigned int abs_obvious(int v) { return (v < 0) ? -(unsigned int)v : (unsigned int)v; }
```

### Minimum and maximum without branching

When x < y, `-(x < y)` is all ones and the XOR picks x. Otherwise it picks
y. The quick versions work out `x - y` only once, but need it not to
overflow. In practice the obvious `?:` usually compiles to a conditional move
and wins.

```c
int min_branchless(int x, int y) { return y ^ ((x ^ y) & -(x < y)); }
int max_branchless(int x, int y) { return x ^ ((x ^ y) & -(x < y)); }
/* Only if INT_MIN <= x - y <= INT_MAX (relies on arithmetic >>): */
int min_quick(int x, int y) { return y + ((x - y) & ((x - y) >> (sizeof(int) * CHAR_BIT - 1))); }
int max_quick(int x, int y) { return x - ((x - y) & ((x - y) >> (sizeof(int) * CHAR_BIT - 1))); }
```

### Is v a power of 2

`(v & (v - 1)) == 0` clears the lowest set bit and checks that nothing is
left. It counts 0 as a power of 2.

Replaced: the original excludes 0 with `v && !(v & (v - 1))`, which is 4
operations. With `t = v - 1`, the test `(v ^ t) > t` is 3. For v = 2^k,
`v ^ t` is 2^(k+1) - 1, which is bigger than t. For v = 0, t is all ones and
nothing is bigger. For anything else, `v ^ t` only reaches as high as v's
lowest set bit, which is below v - 1.

```c
bool is_pow2_or_zero(unsigned int v) { return (v & (v - 1)) == 0; }
bool is_pow2(unsigned int v) { unsigned int t = v - 1; return (v ^ t) > t; }
```

## Sign extension

### From a constant bit-width

A signed bit-field does the sign extension for you. In C++ you can make it a
template: `template <typename T, unsigned B> T signextend(const T x) { struct {T x:B;} s; return s.x = x; }`.
Storing an out-of-range value in a signed bit-field is implementation-defined,
but every mainstream compiler simply truncates it.

```c
int sign_extend_5(int x) {
  struct {signed int x:5;} s;
  return s.x = x;
}
```

### From a variable bit-width

With `m = 1 << (b - 1)`, the expression `(x ^ m) - m` flips the sign bit and
then subtracts it back out, which sign-extends. Skip the masking line if the
bits above b are already zero. Valid for 1 <= b < 32.

Fixed (bug 5): the original's faster version is `(x << m) >> m` on a signed
`int`. Shifting a 1 into the sign bit is undefined behaviour, and that
happens for exactly the negative values. Shifting left as `unsigned` avoids
it. The right shift is still the implementation-defined arithmetic shift the
original warns about. Valid for 1 <= b <= 32.

```c
int sign_extend_mask(int x, unsigned b) {
  int const m = 1U << (b - 1);
  x = x & ((1U << b) - 1);
  return (x ^ m) - m;
}
int sign_extend_shift(int x, unsigned b) {
  int const m = CHAR_BIT * sizeof(x) - b;
  return (int)((unsigned int)x << m) >> m;
}
```

### From a variable bit-width in 3 operations

Multiplying by 2^(32-b) moves bit b-1 into the sign bit, and dividing brings
the value back down, now sign-extended. The 4-operation version has its own
divisor table so that b = 1 works. The 3-operation version needs b > 1. Either
way, x must already fit in b bits.

Fixed (bug 4): the original multiplies in `int`, which overflows for every
negative input. Compilers take advantage of that. gcc (even at -O0) and clang
-O2 reduce `(x * m) / m` to plain `x`, so the 3-operation version returns 31
for the 5-bit value 11111 instead of -1. Multiplying as `unsigned` is well
defined. Converting the result back to `int` is implementation-defined, and
every mainstream compiler wraps.

The original also suggests `(x << -b) >> -b`. A negative shift count is
undefined, so it isn't included here. Use `sign_extend_shift` above.

```c
#define M(B) (1U << ((sizeof(int) * CHAR_BIT) - B))
static int const multipliers[] =
{
  0,     M(1),  M(2),  M(3),  M(4),  M(5),  M(6),  M(7),
  M(8),  M(9),  M(10), M(11), M(12), M(13), M(14), M(15),
  M(16), M(17), M(18), M(19), M(20), M(21), M(22), M(23),
  M(24), M(25), M(26), M(27), M(28), M(29), M(30), M(31),
  M(32)
};
static int const divisors[] =
{
  1,    ~M(1),  M(2),  M(3),  M(4),  M(5),  M(6),  M(7),
  M(8),  M(9),  M(10), M(11), M(12), M(13), M(14), M(15),
  M(16), M(17), M(18), M(19), M(20), M(21), M(22), M(23),
  M(24), M(25), M(26), M(27), M(28), M(29), M(30), M(31),
  M(32)
};
#undef M
int sign_extend_muldiv(int x, unsigned b)  { return (int)((unsigned int)x * multipliers[b]) / divisors[b]; }
int sign_extend_muldiv3(int x, unsigned b) { return (int)((unsigned int)x * multipliers[b]) / multipliers[b]; }
```

## Conditional set, negate and merge

### Set or clear bits depending on a flag

If `f` is set, turn on the bits of `m` in `w`, otherwise turn them off. The
second version uses more operations but more of them can run in parallel.

```c
unsigned int cond_set_clear(unsigned int w, unsigned int m, bool f) {
  w ^= (-f ^ w) & m;
  return w;
}
unsigned int cond_set_clear_superscalar(unsigned int w, unsigned int m, bool f) {
  return (w & ~m) | (-f & m);
}
```

### Negate depending on a flag

In the first version `f ^ (f - 1)` is either +1 or -1. The second uses the
fact that XOR with all ones, plus one, negates a two's-complement number.
Like `-v`, neither can negate `INT_MIN`.

```c
int negate_unless(int v, bool fDontNegate) { return (fDontNegate ^ (fDontNegate - 1)) * v; }
int negate_if(int v, bool fNegate)         { return (v ^ -fNegate) + fNegate; }
```

### Merge bits from two values by a mask

Takes bits from `b` where `mask` is 1 and from `a` elsewhere. It's one
operation shorter than `(a & ~mask) | (b & mask)`.

```c
unsigned int merge_bits(unsigned int a, unsigned int b, unsigned int mask) { return a ^ ((a ^ b) & mask); }
```

## Counting bits set

### Naive and Kernighan

The naive loop goes once per bit up to the highest set one. Kernighan's goes
once per set bit, since `v & (v - 1)` clears the lowest. (Wegner published it
first, in 1960.)

```c
unsigned int popcount_naive(unsigned int v) {
  unsigned int c;
  for (c = 0; v; v >>= 1) { c += v & 1; }
  return c;
}
unsigned int popcount_kernighan(unsigned int v) {
  unsigned int c;
  for (c = 0; v; c++) { v &= v - 1; }
  return c;
}
```

### Lookup table

The table holds the count for each byte value. The macros build it, or you
can fill it with `T[i] = (i & 1) + T[i / 2]`.

```c
static const unsigned char BitsSetTable256[256] =
{
#   define B2(n) n,     n+1,     n+1,     n+2
#   define B4(n) B2(n), B2(n+1), B2(n+1), B2(n+2)
#   define B6(n) B4(n), B4(n+1), B4(n+1), B4(n+2)
    B6(0), B6(1), B6(1), B6(2)
};
unsigned int popcount_table(unsigned int v) {
  return BitsSetTable256[v & 0xff] +
         BitsSetTable256[(v >> 8) & 0xff] +
         BitsSetTable256[(v >> 16) & 0xff] +
         BitsSetTable256[v >> 24];
}
unsigned int popcount_table_bytes(unsigned int v) {
  unsigned char * p = (unsigned char *) &v;
  return BitsSetTable256[p[0]] + BitsSetTable256[p[1]] + BitsSetTable256[p[2]] + BitsSetTable256[p[3]];
}
```

### With 64-bit multiply and modulus

A 64-bit multiply lays out spaced copies of the bits, the mask keeps one bit
in each group, and the `%` adds the groups up. That takes 3 operations for 14
bits, or 10 for 24.

Replaced for full 32-bit words: the original's 15-operation version is
swapped for HAKMEM item 169 (Beeler, Gosper and Schroeppel, 1972). It counts
bits within 3-bit fields, adds neighbouring fields into 6-bit fields, and sums
those with `% 63`, which works because 64 leaves remainder 1 mod 63. It's 10
operations, compiles to about half the instructions, and runs about twice as
fast. If you don't have a hardware popcount, the parallel "best" method below
is still quicker.

```c
unsigned int popcount14(unsigned int v) { return (v * 0x200040008001ULL & 0x111111111111111ULL) % 0xf; }
unsigned int popcount24(unsigned int v) {
  unsigned int c;
  c =  ((v & 0xfff) * 0x1001001001001ULL & 0x84210842108421ULL) % 0x1f;
  c += (((v & 0xfff000) >> 12) * 0x1001001001001ULL & 0x84210842108421ULL) % 0x1f;
  return c;
}
unsigned int popcount32_hakmem(unsigned int v) {
  unsigned int n;
  n = (v >> 1) & 033333333333;
  v = v - n;
  n = (n >> 1) & 033333333333;
  v = v - n;
  return ((v + (v >> 3)) & 030707070707) % 63;
}
```

### In parallel

Each step adds neighbouring fields that are twice as wide as before. The
"best" version stops at bytes and adds them up with one multiply, for 12
operations. Its generic form works for unsigned types up to 128 bits; here
it's shown for `uint64_t`.

```c
unsigned int popcount_parallel(unsigned int v) {
  static const int S[] = {1, 2, 4, 8, 16};
  static const int B[] = {0x55555555, 0x33333333, 0x0F0F0F0F, 0x00FF00FF, 0x0000FFFF};
  unsigned int c;
  c = v - ((v >> 1) & B[0]);
  c = ((c >> S[1]) & B[1]) + (c & B[1]);
  c = ((c >> S[2]) + c) & B[2];
  c = ((c >> S[3]) + c) & B[3];
  c = ((c >> S[4]) + c) & B[4];
  return c;
}
unsigned int popcount_best(unsigned int v) {
  v = v - ((v >> 1) & 0x55555555);
  v = (v & 0x33333333) + ((v >> 2) & 0x33333333);
  return ((v + (v >> 4) & 0xF0F0F0F) * 0x1010101) >> 24;
}
typedef uint64_t T;
unsigned int popcount_generic(T v) {
  v = v - ((v >> 1) & (T)~(T)0/3);
  v = (v & (T)~(T)0/15*3) + ((v >> 2) & (T)~(T)0/15*3);
  v = (v + (v >> 4)) & (T)~(T)0/255*15;
  return (T)(v * ((T)~(T)0/255)) >> (sizeof(T) - 1) * CHAR_BIT;
}
```

### Rank: bits set from the MSB up to a position

Shift away everything after `pos` (1 to 64), then count what's left.

Fixed (bug 11): the original works on `uint64_t` but builds its masks from
`~0UL`. On 64-bit Windows and on 32-bit targets `unsigned long` is 32 bits,
so the masks lose their top halves and `rank(~0, 64)` returns 0. `~0ULL` is 64
bits on every platform.

```c
uint64_t rank(uint64_t v, unsigned int pos) {
  uint64_t r;
  r = v >> (sizeof(v) * CHAR_BIT - pos);
  r = r - ((r >> 1) & ~0ULL/3);
  r = (r & ~0ULL/5) + ((r >> 2) & ~0ULL/5);
  r = (r + (r >> 4)) & ~0ULL/17;
  r = (r * (~0ULL/255)) >> ((sizeof(v) - 1) * CHAR_BIT);
  return r;
}
```

### Select: position of the bit with a given rank

This is the reverse of rank, counting from the most significant bit. It keeps
every stage of the parallel count, then narrows in on the position without
branching, halving the range each step. If `r` is bigger than the number of
set bits, it returns 64.

Fixed (bug 11): `~0UL` becomes `~0ULL`, as in rank.

Fixed (bug 1): the original says to swap in `if` statements if branching is
cheap. That breaks it, because the first count `t` isn't masked to 8 bits: the
top lane's count spills into bits 16 to 23, and `if (r > t)` then compares the
wrong value. `select_bit_branchy` adds the `& 0xff`.

```c
unsigned int select_bit(uint64_t v, unsigned int r) {
  unsigned int s;
  uint64_t a, b, c, d;
  unsigned int t;
  a =  v - ((v >> 1) & ~0ULL/3);
  b = (a & ~0ULL/5) + ((a >> 2) & ~0ULL/5);
  c = (b + (b >> 4)) & ~0ULL/0x11;
  d = (c + (c >> 8)) & ~0ULL/0x101;
  t = (d >> 32) + (d >> 48);
  s  = 64;
  s -= ((t - r) & 256) >> 3; r -= (t & ((t - r) >> 8));
  t  = (d >> (s - 16)) & 0xff;
  s -= ((t - r) & 256) >> 4; r -= (t & ((t - r) >> 8));
  t  = (c >> (s - 8)) & 0xf;
  s -= ((t - r) & 256) >> 5; r -= (t & ((t - r) >> 8));
  t  = (b >> (s - 4)) & 0x7;
  s -= ((t - r) & 256) >> 6; r -= (t & ((t - r) >> 8));
  t  = (a >> (s - 2)) & 0x3;
  s -= ((t - r) & 256) >> 7; r -= (t & ((t - r) >> 8));
  t  = (v >> (s - 1)) & 0x1;
  s -= ((t - r) & 256) >> 8;
  s = 65 - s;
  return s;
}
unsigned int select_bit_branchy(uint64_t v, unsigned int r) {
  unsigned int s;
  uint64_t a, b, c, d;
  unsigned int t;
  a =  v - ((v >> 1) & ~0ULL/3);
  b = (a & ~0ULL/5) + ((a >> 2) & ~0ULL/5);
  c = (b + (b >> 4)) & ~0ULL/0x11;
  d = (c + (c >> 8)) & ~0ULL/0x101;
  t = ((d >> 32) + (d >> 48)) & 0xff;
  s  = 64;
  if (r > t) {s -= 32; r -= t;}
  t  = (d >> (s - 16)) & 0xff;
  if (r > t) {s -= 16; r -= t;}
  t  = (c >> (s - 8)) & 0xf;
  if (r > t) {s -= 8; r -= t;}
  t  = (b >> (s - 4)) & 0x7;
  if (r > t) {s -= 4; r -= t;}
  t  = (a >> (s - 2)) & 0x3;
  if (r > t) {s -= 2; r -= t;}
  t  = (v >> (s - 1)) & 0x1;
  if (r > t) s--;
  return 65 - s;
}
```

## Parity

Parity is 1 when an odd number of bits are set.

### Naive and lookup table

The naive loop flips once per set bit. The table version first XORs the
word's halves together down to a single byte.

```c
bool parity_naive(unsigned int v) {
  bool parity = false;
  while (v) { parity = !parity; v = v & (v - 1); }
  return parity;
}
static const bool ParityTable256[256] =
{
#   define P2(n) n, n^1, n^1, n
#   define P4(n) P2(n), P2(n^1), P2(n^1), P2(n)
#   define P6(n) P4(n), P4(n^1), P4(n^1), P4(n)
    P6(0), P6(1), P6(1), P6(0)
};
bool parity_byte_table(unsigned char b) { return ParityTable256[b]; }
bool parity_table(unsigned int v) {
  v ^= v >> 16;
  v ^= v >> 8;
  return ParityTable256[v & 0xff];
}
bool parity_table_bytes(unsigned int v) {
  unsigned char * p = (unsigned char *) &v;
  return ParityTable256[p[0] ^ p[1] ^ p[2] ^ p[3]];
}
```

### Parity of a byte with 64-bit multiply and modulus

```c
bool parity_byte_mulmod(unsigned char b) {
  return (((b * 0x0101010101010101ULL) & 0x8040201008040201ULL) % 0x1FF) & 1;
}
```

### Parity of a word with a multiply

Replaced, for both 32 and 64 bits. The original folds the bits rightwards,
multiplies, and then needs both `>> 28` and `& 1` (8 operations). If you fold
to the left instead, each nibble's parity lands in that nibble's top bit. The
multiply then adds all of those into the top bit of the word, and anything
above it is simply lost off the end, so the `& 1` goes away. That makes 7
operations. Nothing can carry into the top bit: the partial sum landing at
bit 4k+3 is at most k+1, which fits in the four bits below the next group.

```c
bool parity_mul(uint32_t v) {
  v ^= v << 1;
  v ^= v << 2;
  return ((v & 0x88888888U) * 0x11111111U) >> 31;
}
bool parity_mul64(uint64_t v) {
  v ^= v << 1;
  v ^= v << 2;
  return ((v & 0x8888888888888888ULL) * 0x1111111111111111ULL) >> 63;
}
```

### In parallel

Fold down to one nibble, then use 0x6996 as a 16-entry parity table. Drop the
first two lines and you have a 5-operation version for bytes.

```c
bool parity_parallel(unsigned int v) {
  v ^= v >> 16;
  v ^= v >> 8;
  v ^= v >> 4;
  v &= 0xf;
  return (0x6996 >> v) & 1;
}
bool parity_byte_parallel(unsigned char b) {
  unsigned int v = b;
  v ^= v >> 4;
  v &= 0xf;
  return (0x6996 >> v) & 1;
}
```

## Swapping

### Values, without a temporary

The subtraction macro checks that `a` and `b` aren't the same object, and you
should give it unsigned values if overflow traps. The plain XOR swap zeroes the
value when both arguments are the same object, as in `SWAP(a[i], a[j])` with
i == j. The last two avoid that: one compares the values, the other reuses
`a ^ b`.

```c
#define SWAP_SUB(a, b) ((&(a) == &(b)) || \
                        (((a) -= (b)), ((b) += (a)), ((a) = (b) - (a))))
#define SWAP_XOR(a, b) (((a) ^= (b)), ((b) ^= (a)), ((a) ^= (b)))
#define SWAP_XOR_SAFE(a, b) (((a) == (b)) || (((a) ^= (b)), ((b) ^= (a)), ((a) ^= (b))))
#define SWAP_XOR_FURUSETH(a, b) (((a) ^ (b)) && ((b) ^= (a) ^= (b), (a) ^= (b)))
```

### Ranges of bits

Swaps the n-bit fields at positions i and j. They must not overlap, and n must
be under 32. `x` marks where the two fields differ, and XORing it back in at
both positions swaps them.

```c
unsigned int swap_bits(unsigned int b, unsigned int i, unsigned int j, unsigned int n) {
  unsigned int x = ((b >> i) ^ (b >> j)) & ((1U << n) - 1);
  return b ^ ((x << i) | (x << j));
}
```

## Reversing bits

### The obvious way

Shift bits out of v and into r until v is empty, then shift r by however
many places are left.

```c
unsigned int reverse_obvious(unsigned int v) {
  unsigned int r = v;
  int s = sizeof(v) * CHAR_BIT - 1;
  for (v >>= 1; v; v >>= 1) {
    r <<= 1;
    r |= v & 1;
    s--;
  }
  r <<= s;
  return r;
}
```

### By lookup table

Reverse each byte with a table, then put the bytes in reverse order.

Fixed (bug 7): in the original's first option the table value is promoted to
a signed `int` before `<< 24`. Shifting 128 or more that far overflows, and
that happens for every odd v. Casting to `unsigned int` first fixes it. The
second option stores bytes and never had the problem.

```c
static const unsigned char BitReverseTable256[256] =
{
#   define R2(n)     n,     n + 2*64,     n + 1*64,     n + 3*64
#   define R4(n) R2(n), R2(n + 2*16), R2(n + 1*16), R2(n + 3*16)
#   define R6(n) R4(n), R4(n + 2*4 ), R4(n + 1*4 ), R4(n + 3*4 )
    R6(0), R6(2), R6(1), R6(3)
};
unsigned int reverse_table(unsigned int v) {
  return ((unsigned int)BitReverseTable256[v & 0xff] << 24) |
         ((unsigned int)BitReverseTable256[(v >> 8) & 0xff] << 16) |
         ((unsigned int)BitReverseTable256[(v >> 16) & 0xff] << 8) |
         ((unsigned int)BitReverseTable256[(v >> 24) & 0xff]);
}
unsigned int reverse_table_bytes(unsigned int v) {
  unsigned int c;
  unsigned char * p = (unsigned char *) &v;
  unsigned char * q = (unsigned char *) &c;
  q[3] = BitReverseTable256[p[0]];
  q[2] = BitReverseTable256[p[1]];
  q[1] = BitReverseTable256[p[2]];
  q[0] = BitReverseTable256[p[3]];
  return c;
}
```

### A single byte

The 3-operation version uses a 64-bit multiply to make five copies of the
byte, masks out each bit at its reversed position within a 10-bit group, and
then adds the groups with `% 1023` (Schroeppel, in HAKMEM). The 4-operation
version gathers the bits with a second multiply instead of dividing. The
7-operation one sticks to 32-bit arithmetic.

```c
unsigned char reverse_byte_3ops(unsigned char b) { return (b * 0x0202020202ULL & 0x010884422010ULL) % 1023; }
unsigned char reverse_byte_4ops(unsigned char b) { return ((b * 0x80200802ULL) & 0x0884422110ULL) * 0x0101010101ULL >> 32; }
unsigned char reverse_byte_7ops(unsigned char b) {
  return ((b * 0x0802LU & 0x22110LU) | (b * 0x8020LU & 0x88440LU)) * 0x10101LU >> 16;
}
```

### In parallel

Swap odd and even bits, then pairs, nibbles, bytes and finally the two halves.
The loop version works out each mask as it goes, so it needs no constants,
at the cost of a few extra operations.

```c
unsigned int reverse_parallel(unsigned int v) {
  v = ((v >> 1) & 0x55555555) | ((v & 0x55555555) << 1);
  v = ((v >> 2) & 0x33333333) | ((v & 0x33333333) << 2);
  v = ((v >> 4) & 0x0F0F0F0F) | ((v & 0x0F0F0F0F) << 4);
  v = ((v >> 8) & 0x00FF00FF) | ((v & 0x00FF00FF) << 8);
  v = ( v >> 16             ) | ( v               << 16);
  return v;
}
unsigned int reverse_parallel_loop(unsigned int v) {
  unsigned int s = sizeof(v) * CHAR_BIT;
  unsigned int mask = ~0U;
  while ((s >>= 1) > 0) {
    mask ^= (mask << s);
    v = ((v >> s) & mask) | ((v << s) & ~mask);
  }
  return v;
}
```

## Modulus division

### By 1 << s

```c
unsigned int mod_pow2(unsigned int n, unsigned int s) {
  const unsigned int d = 1U << s;
  return n & (d - 1);
}
```

### By (1 << s) - 1, without division

With d = 2^s - 1, 2^s leaves remainder 1, so adding up n's base-2^s digits
keeps the same remainder. Repeat until the sum is no bigger than d, and turn
d itself into 0. s must be positive.

Fixed (bug 9): `(1 << s) - 1` with a signed 1 overflows when s = 31. It's
`1U` here, the same fix the original made in "swapping individual bits".

```c
unsigned int mod_mersenne(unsigned int n, const unsigned int s) {
  const unsigned int d = (1U << s) - 1;
  unsigned int m;
  for (m = n; n > d; n = m) {
    for (m = 0; n; n >>= s) { m += n & d; }
  }
  m = m == d ? 0 : m;
  return m;
}
```

### By (1 << s) - 1, in parallel

The same digit sum, done in parallel: add pairs of digits first, then keep
folding the halves together, with shift amounts from `Q` and masks from `R`.
That takes O(lg N) operations. 32-bit words only.

Fixed (bug 3): with s = 1 the folding loop runs off the end of its six-entry
row and then off the end of the `Q` and `R` arrays altogether. Since n mod 1
is always 0, s = 1 now returns straight away. For every s from 2 to 31 and
every 32-bit n, six steps are enough.

Fixed (bug 9): `1U`, as above.

Fixed (bug 2): the original also offers a branch-free ending,
`m & -((signed)(m - d) >> s)`, which gives `m & 1` whenever m < d. Without the
minus it works, and that's the version in the comment below.

```c
static const unsigned int M[] =
{
  0x00000000, 0x55555555, 0x33333333, 0xc71c71c7,
  0x0f0f0f0f, 0xc1f07c1f, 0x3f03f03f, 0xf01fc07f,
  0x00ff00ff, 0x07fc01ff, 0x3ff003ff, 0xffc007ff,
  0xff000fff, 0xfc001fff, 0xf0003fff, 0xc0007fff,
  0x0000ffff, 0x0001ffff, 0x0003ffff, 0x0007ffff,
  0x000fffff, 0x001fffff, 0x003fffff, 0x007fffff,
  0x00ffffff, 0x01ffffff, 0x03ffffff, 0x07ffffff,
  0x0fffffff, 0x1fffffff, 0x3fffffff, 0x7fffffff
};
static const unsigned int Q[][6] =
{
  { 0,  0,  0,  0,  0,  0}, {16,  8,  4,  2,  1,  1}, {16,  8,  4,  2,  2,  2},
  {15,  6,  3,  3,  3,  3}, {16,  8,  4,  4,  4,  4}, {15,  5,  5,  5,  5,  5},
  {12,  6,  6,  6 , 6,  6}, {14,  7,  7,  7,  7,  7}, {16,  8,  8,  8,  8,  8},
  { 9,  9,  9,  9,  9,  9}, {10, 10, 10, 10, 10, 10}, {11, 11, 11, 11, 11, 11},
  {12, 12, 12, 12, 12, 12}, {13, 13, 13, 13, 13, 13}, {14, 14, 14, 14, 14, 14},
  {15, 15, 15, 15, 15, 15}, {16, 16, 16, 16, 16, 16}, {17, 17, 17, 17, 17, 17},
  {18, 18, 18, 18, 18, 18}, {19, 19, 19, 19, 19, 19}, {20, 20, 20, 20, 20, 20},
  {21, 21, 21, 21, 21, 21}, {22, 22, 22, 22, 22, 22}, {23, 23, 23, 23, 23, 23},
  {24, 24, 24, 24, 24, 24}, {25, 25, 25, 25, 25, 25}, {26, 26, 26, 26, 26, 26},
  {27, 27, 27, 27, 27, 27}, {28, 28, 28, 28, 28, 28}, {29, 29, 29, 29, 29, 29},
  {30, 30, 30, 30, 30, 30}, {31, 31, 31, 31, 31, 31}
};
static const unsigned int R[][6] =
{
  {0x00000000, 0x00000000, 0x00000000, 0x00000000, 0x00000000, 0x00000000},
  {0x0000ffff, 0x000000ff, 0x0000000f, 0x00000003, 0x00000001, 0x00000001},
  {0x0000ffff, 0x000000ff, 0x0000000f, 0x00000003, 0x00000003, 0x00000003},
  {0x00007fff, 0x0000003f, 0x00000007, 0x00000007, 0x00000007, 0x00000007},
  {0x0000ffff, 0x000000ff, 0x0000000f, 0x0000000f, 0x0000000f, 0x0000000f},
  {0x00007fff, 0x0000001f, 0x0000001f, 0x0000001f, 0x0000001f, 0x0000001f},
  {0x00000fff, 0x0000003f, 0x0000003f, 0x0000003f, 0x0000003f, 0x0000003f},
  {0x00003fff, 0x0000007f, 0x0000007f, 0x0000007f, 0x0000007f, 0x0000007f},
  {0x0000ffff, 0x000000ff, 0x000000ff, 0x000000ff, 0x000000ff, 0x000000ff},
  {0x000001ff, 0x000001ff, 0x000001ff, 0x000001ff, 0x000001ff, 0x000001ff},
  {0x000003ff, 0x000003ff, 0x000003ff, 0x000003ff, 0x000003ff, 0x000003ff},
  {0x000007ff, 0x000007ff, 0x000007ff, 0x000007ff, 0x000007ff, 0x000007ff},
  {0x00000fff, 0x00000fff, 0x00000fff, 0x00000fff, 0x00000fff, 0x00000fff},
  {0x00001fff, 0x00001fff, 0x00001fff, 0x00001fff, 0x00001fff, 0x00001fff},
  {0x00003fff, 0x00003fff, 0x00003fff, 0x00003fff, 0x00003fff, 0x00003fff},
  {0x00007fff, 0x00007fff, 0x00007fff, 0x00007fff, 0x00007fff, 0x00007fff},
  {0x0000ffff, 0x0000ffff, 0x0000ffff, 0x0000ffff, 0x0000ffff, 0x0000ffff},
  {0x0001ffff, 0x0001ffff, 0x0001ffff, 0x0001ffff, 0x0001ffff, 0x0001ffff},
  {0x0003ffff, 0x0003ffff, 0x0003ffff, 0x0003ffff, 0x0003ffff, 0x0003ffff},
  {0x0007ffff, 0x0007ffff, 0x0007ffff, 0x0007ffff, 0x0007ffff, 0x0007ffff},
  {0x000fffff, 0x000fffff, 0x000fffff, 0x000fffff, 0x000fffff, 0x000fffff},
  {0x001fffff, 0x001fffff, 0x001fffff, 0x001fffff, 0x001fffff, 0x001fffff},
  {0x003fffff, 0x003fffff, 0x003fffff, 0x003fffff, 0x003fffff, 0x003fffff},
  {0x007fffff, 0x007fffff, 0x007fffff, 0x007fffff, 0x007fffff, 0x007fffff},
  {0x00ffffff, 0x00ffffff, 0x00ffffff, 0x00ffffff, 0x00ffffff, 0x00ffffff},
  {0x01ffffff, 0x01ffffff, 0x01ffffff, 0x01ffffff, 0x01ffffff, 0x01ffffff},
  {0x03ffffff, 0x03ffffff, 0x03ffffff, 0x03ffffff, 0x03ffffff, 0x03ffffff},
  {0x07ffffff, 0x07ffffff, 0x07ffffff, 0x07ffffff, 0x07ffffff, 0x07ffffff},
  {0x0fffffff, 0x0fffffff, 0x0fffffff, 0x0fffffff, 0x0fffffff, 0x0fffffff},
  {0x1fffffff, 0x1fffffff, 0x1fffffff, 0x1fffffff, 0x1fffffff, 0x1fffffff},
  {0x3fffffff, 0x3fffffff, 0x3fffffff, 0x3fffffff, 0x3fffffff, 0x3fffffff},
  {0x7fffffff, 0x7fffffff, 0x7fffffff, 0x7fffffff, 0x7fffffff, 0x7fffffff}
};
unsigned int mod_mersenne_parallel(unsigned int n, const unsigned int s) {
  const unsigned int d = (1U << s) - 1;
  unsigned int m;
  if (s == 1) return 0;
  m = (n & M[s]) + ((n >> s) & M[s]);
  for (const unsigned int * q = &Q[s][0], * r = &R[s][0]; m > d; q++, r++) {
    m = (m >> *q) + (m & *r);
  }
  m = m == d ? 0 : m;   /* or, branch-free: m = m & ((signed)(m - d) >> s); */
  return m;
}
```

## Log base 2

The integer log base 2 of v is the position of its highest set bit. Unless
noted otherwise, v must be greater than 0.

### The obvious way

```c
unsigned int log2_obvious(unsigned int v) {
  unsigned int r = 0;
  while (v >>= 1) { r++; }
  return r;
}
```

### With a 64-bit IEEE float

Replaced: the original builds a double in a union, with two stores, subtracts
2^52, then shifts and removes the bias, for 5 operations. Any 32-bit integer
fits exactly in a double, though, so a plain conversion does the job, and the
exponent field is then 1023 + log2(v). That's 4 operations, and in testing it
was about a quarter faster. It assumes IEEE-754 doubles and the same byte
order for floats and integers, as the original does.

```c
unsigned int log2_double(uint32_t v) {
  double d = (double)v;
  uint64_t bits;
  memcpy(&bits, &d, sizeof bits);
  return (unsigned int)(bits >> 52) - 1023;
}
```

### With a lookup table

Find the highest byte that isn't zero, then look up that byte's log. The
first version is tuned for evenly spread outputs, the second for evenly
spread inputs. You can build the table with `T[0] = T[1] = 0; T[i] = 1 + T[i / 2]`.
Entry 0 holds -1, so log2(0) returns -1, which as an `unsigned` is `UINT_MAX`.

Fixed (bug 13): the original table is plain `char`. On ARM and PowerPC Linux,
plain `char` is unsigned, so entry 0 ends up as 255. Declaring it `signed
char` fixes that.

```c
static const signed char LogTable256[256] =
{
#define LT(n) n, n, n, n, n, n, n, n, n, n, n, n, n, n, n, n
    -1, 0, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3,
    LT(4), LT(5), LT(5), LT(6), LT(6), LT(6), LT(6),
    LT(7), LT(7), LT(7), LT(7), LT(7), LT(7), LT(7), LT(7)
};
unsigned int log2_table(unsigned int v) {
  unsigned r;
  register unsigned int t, tt;
  if ((tt = v >> 16)) {
    r = (t = tt >> 8) ? 24 + LogTable256[t] : 16 + LogTable256[tt];
  } else {
    r = (t = v >> 8) ? 8 + LogTable256[t] : LogTable256[v];
  }
  return r;
}
unsigned int log2_table_uniform_inputs(unsigned int v) {
  unsigned r;
  register unsigned int tt;
  if ((tt = v >> 24))      { r = 24 + LogTable256[tt]; }
  else if ((tt = v >> 16)) { r = 16 + LogTable256[tt]; }
  else if ((tt = v >> 8))  { r = 8 + LogTable256[tt]; }
  else                     { r = LogTable256[v]; }
  return r;
}
```

### In O(lg N) operations

This is a binary search on the bit position: test the top 16 bits, then 8, 4,
2 and 1, and shift down whenever a test hits. The branch-free version turns
each comparison into a shift amount. The last version only works if v is
already a power of 2.

```c
unsigned int log2_lg(unsigned int v) {
  const unsigned int b[] = {0x2, 0xC, 0xF0, 0xFF00, 0xFFFF0000};
  const unsigned int S[] = {1, 2, 4, 8, 16};
  int i;
  register unsigned int r = 0;
  for (i = 4; i >= 0; i--) {
    if (v & b[i]) { v >>= S[i]; r |= S[i]; }
  }
  return r;
}
unsigned int log2_lg_branchless(unsigned int v) {
  register unsigned int r;
  register unsigned int shift;
  r =     (v > 0xFFFF) << 4; v >>= r;
  shift = (v > 0xFF  ) << 3; v >>= shift; r |= shift;
  shift = (v > 0xF   ) << 2; v >>= shift; r |= shift;
  shift = (v > 0x3   ) << 1; v >>= shift; r |= shift;
                                          r |= (v >> 1);
  return r;
}
unsigned int log2_of_pow2_masks(unsigned int v) {
  static const unsigned int b[] = {0xAAAAAAAA, 0xCCCCCCCC, 0xF0F0F0F0, 0xFF00FF00, 0xFFFF0000};
  int i;
  register unsigned int r = (v & b[0]) != 0;
  for (i = 4; i > 0; i--) { r |= ((v & b[i]) != 0) << i; }
  return r;
}
```

### Multiply and lookup

Copy the top bit into every bit below it, which gives 2^(k+1) - 1. Multiplying
by the right constant leaves a different 5-bit pattern in the top bits for
each k, and that indexes a small table. If v is already a power of 2, you can
skip the copying and use a different constant and table.

```c
static const int MultiplyDeBruijnBitPosition[32] =
{
  0, 9, 1, 10, 13, 21, 2, 29, 11, 14, 16, 18, 22, 25, 3, 30,
  8, 12, 20, 28, 15, 17, 24, 7, 19, 27, 23, 6, 26, 5, 4, 31
};
int log2_debruijn(uint32_t v) {
  v |= v >> 1;
  v |= v >> 2;
  v |= v >> 4;
  v |= v >> 8;
  v |= v >> 16;
  return MultiplyDeBruijnBitPosition[(uint32_t)(v * 0x07C4ACDDU) >> 27];
}
static const int MultiplyDeBruijnBitPosition2[32] =
{
  0, 1, 28, 2, 29, 14, 24, 3, 30, 22, 20, 15, 25, 17, 4, 8,
  31, 27, 13, 23, 21, 19, 16, 7, 26, 12, 18, 6, 11, 5, 10, 9
};
int log2_of_pow2_debruijn(uint32_t v) {
  return MultiplyDeBruijnBitPosition2[(uint32_t)(v * 0x077CB531U) >> 27];
}
```

## Log base 10

### Using log base 2

1233/4096 is close to 1/log2(10), so `(log2(v) + 1) * 1233 >> 12` estimates
log10(v). The estimate can be one too high; subtract 1 if v is less than the
matching power of 10. Any log2 method works. The original writes it as
`t = (IntegerLogBase2(v) + 1) * 1233 >> 12; r = t - (v < PowersOf10[t]);`.

Replaced: this version uses the double-based log2 from above and folds the
`+ 1` into the exponent bias (-1023 + 1 = -1022). That's 9 operations with no
extra tables. The original mentions getting down to 9 operations by building
four 256-entry tables. Compared with the original's formula plus its table
log2, this is 15 to 30% faster. v must be greater than 0.

```c
static unsigned int const PowersOf10[] =
    {1, 10, 100, 1000, 10000, 100000,
     1000000, 10000000, 100000000, 1000000000};
unsigned int log10_int(uint32_t v) {
  double d = (double)v;
  uint64_t bits;
  unsigned int t;
  memcpy(&bits, &d, sizeof bits);
  t = (((unsigned int)(bits >> 52) - 1022) * 1233) >> 12;
  return t - (v < PowersOf10[t]);
}
```

### The obvious way

With evenly spread 32-bit inputs, most values stop at the first or second
comparison.

```c
unsigned int log10_obvious(unsigned int v) {
  return (v >= 1000000000) ? 9 : (v >= 100000000) ? 8 : (v >= 10000000) ? 7 :
         (v >= 1000000) ? 6 : (v >= 100000) ? 5 : (v >= 10000) ? 4 :
         (v >= 1000) ? 3 : (v >= 100) ? 2 : (v >= 10) ? 1 : 0;
}
```

## Log base 2 of a float

### Of a 32-bit IEEE float

For a normal float, the exponent field holds floor(log2(v)) + 127. Subnormals
have an exponent field of 0, so the second version falls back to the lookup
table on the mantissa. v must be positive and finite, and for the first
version also normal.

Fixed (bug 14, documentation only): the original says the result is
`int(log2(v))`. In C, converting to `int` rounds toward zero, but the code
returns the floor. For 0 < v < 1 those disagree: v = 0.3 gives -2, while
`int(log2(0.3))` is -1. The code is right, and the result is floor(log2(v)).

```c
int log2_float(float v) {
  int c;
  memcpy(&c, &v, sizeof c);
  c = (c >> 23) - 127;
  return c;
}
int log2_float_subnormal(float v) {
  int c;
  int x;
  memcpy(&x, &v, sizeof x);
  c = x >> 23;
  if (c) {
    c -= 127;
  } else {
    register unsigned int t;
    if ((t = x >> 16)) {
      c = LogTable256[t] - 133;
    } else {
      c = (t = x >> 8) ? LogTable256[t] - 141 : LogTable256[x] - 149;
    }
  }
  return c;
}
```

### Of the 2^r-th root of a 32-bit IEEE float

Removing the bias and shifting right by r divides the log by 2^r. The result
is floor(log2(v) / 2^r) for normal v > 0. The original says `int(...)` here
too, which goes wrong for negative results in the same way (bug 14). For
v < 1 it relies on an arithmetic `>>`.

```c
int log2_float_root(float v, int r) {
  int c;
  memcpy(&c, &v, sizeof c);
  c = ((((c - 0x3f800000) >> r) + 0x3f800000) >> 23) - 127;
  return c;
}
```

## Counting trailing zeros

### Linearly

`(v ^ (v - 1)) >> 1` turns the trailing zeros into ones and clears everything
else, and then the loop counts them. On random input that's about one pass
on average. Returns 32 for v = 0.

```c
unsigned int ctz_linear(unsigned int v) {
  unsigned int c;
  if (v) {
    v = (v ^ (v - 1)) >> 1;
    for (c = 0; v; c++) { v >>= 1; }
  } else {
    c = CHAR_BIT * sizeof(v);
  }
  return c;
}
```

### In parallel

Isolate the lowest set bit, then find its position with five mask tests.
Returns 32 for v = 0.

Fixed (bug 10): the original has `v &= -signed(v)`, which only compiles as
C++, and overflows there for v = 0x80000000. `-v` on an unsigned value is
fine in both languages.

```c
unsigned int ctz_parallel(unsigned int v) {
  unsigned int c = 32;
  v &= -v;
  if (v) c--;
  if (v & 0x0000FFFF) c -= 16;
  if (v & 0x00FF00FF) c -= 8;
  if (v & 0x0F0F0F0F) c -= 4;
  if (v & 0x33333333) c -= 2;
  if (v & 0x55555555) c -= 1;
  return c;
}
```

### Binary search

Each step halves the range. Odd numbers, half of all inputs, exit on the first
test. Returns 31 for v = 0.

```c
unsigned int ctz_binary_search(unsigned int v) {
  unsigned int c;
  if (v & 0x1) {
    c = 0;
  } else {
    c = 1;
    if ((v & 0xffff) == 0) { v >>= 16; c += 16; }
    if ((v & 0xff) == 0)   { v >>= 8;  c += 8; }
    if ((v & 0xf) == 0)    { v >>= 4;  c += 4; }
    if ((v & 0x3) == 0)    { v >>= 2;  c += 2; }
    c -= v & 0x1;
  }
  return c;
}
```

### By converting to a float

Converted to a float, the lowest set bit has its position as the exponent.
Returns -127 for v = 0.

```c
int ctz_float(unsigned int v) {
  float f = (float)(v & -v);
  uint32_t u;
  memcpy(&u, &f, sizeof u);
  return (int)(u >> 23) - 0x7f;
}
```

### Modulus and lookup

The 32 single-bit values and 0 all leave different remainders mod 37, so a
37-entry table can map them back to positions. (Reiser had this first.)
Returns 32 for v = 0.

```c
static const int Mod37BitPosition[] =
{
  32, 0, 1, 26, 2, 23, 27, 0, 3, 16, 24, 30, 28, 11, 0, 13, 4,
  7, 17, 0, 25, 22, 31, 15, 29, 10, 12, 6, 0, 21, 14, 9, 5,
  20, 8, 19, 18
};
int ctz_mod37(unsigned int v) { return Mod37BitPosition[(-v & v) % 37]; }
```

### Multiply and lookup

Multiply the lowest set bit by the de Bruijn constant 0x077CB531 and the top
five bits identify it (Leiserson, Prokop and Randall). It uses the same table
as the power-of-2 log above. Returns 0 for v = 0.

```c
int ctz_debruijn(uint32_t v) {
  return MultiplyDeBruijnBitPosition2[((uint32_t)((v & -v) * 0x077CB531U)) >> 27];
}
```

## Rounding up to a power of 2

### By float casting

Convert v to a `float`, read the exponent, and double the result if it's
still below v. That's 8 operations and covers every v up to 2^31, with 0 and
1 both giving 1. The quick version takes 6 operations but only works for
1 < v < 2^25, because `(float)(v - 1)` gets rounded beyond that.

Converting to `double` instead makes the quick version exact up to 2^31, in
7 operations. I left it out because it wasn't measurably faster.

```c
unsigned int round_up_pow2_float(unsigned int v) {
  unsigned int r;
  if (v > 1) {
    float f = (float)v;
    unsigned int fb;
    memcpy(&fb, &f, sizeof fb);
    unsigned int const t = 1U << ((fb >> 23) - 0x7f);
    r = t << (t < v);
  } else {
    r = 1;
  }
  return r;
}
unsigned int round_up_pow2_float_quick(unsigned int v) {
  float f = (float)(v - 1);
  unsigned int fb;
  memcpy(&fb, &f, sizeof fb);
  return 1U << ((fb >> 23) - 126);
}
```

### By copying the top bit down

Subtract one, copy the highest set bit into every bit below it, and add one
back. The first subtraction makes exact powers of 2 come out unchanged. This
takes 12 operations and no floating point. It returns 0 for v = 0, so add
`v += (v == 0)` if that matters.

```c
unsigned int round_up_pow2(unsigned int v) {
  v--;
  v |= v >> 1;
  v |= v >> 2;
  v |= v >> 4;
  v |= v >> 8;
  v |= v >> 16;
  v++;
  return v;
}
```

## Interleaving bits

Interleaving puts the bits of x in the even positions and the bits of y in
the odd positions, giving a Morton number. Points that are close in 2-D
usually end up with close numbers.

### The obvious way

```c
unsigned int morton_obvious(unsigned short x, unsigned short y) {
  unsigned int z = 0;
  for (unsigned i = 0; i < sizeof(x) * CHAR_BIT; i++) {
    z |= (x & 1U << i) << i | (y & 1U << i) << (i + 1);
  }
  return z;
}
```

### By table lookup

The table spreads each byte value out to the even bits.

Fixed (bug 8): the original shifts the table value as a signed `int`, and
`<< 17` overflows when y >= 0x8000. Here the values are cast to `unsigned
int` before shifting. A second table, pre-shifted by one, would save two
operations; with four tables you'd get down to 11.

```c
static const unsigned short MortonTable256[256] =
{
  0x0000, 0x0001, 0x0004, 0x0005, 0x0010, 0x0011, 0x0014, 0x0015,
  0x0040, 0x0041, 0x0044, 0x0045, 0x0050, 0x0051, 0x0054, 0x0055,
  0x0100, 0x0101, 0x0104, 0x0105, 0x0110, 0x0111, 0x0114, 0x0115,
  0x0140, 0x0141, 0x0144, 0x0145, 0x0150, 0x0151, 0x0154, 0x0155,
  0x0400, 0x0401, 0x0404, 0x0405, 0x0410, 0x0411, 0x0414, 0x0415,
  0x0440, 0x0441, 0x0444, 0x0445, 0x0450, 0x0451, 0x0454, 0x0455,
  0x0500, 0x0501, 0x0504, 0x0505, 0x0510, 0x0511, 0x0514, 0x0515,
  0x0540, 0x0541, 0x0544, 0x0545, 0x0550, 0x0551, 0x0554, 0x0555,
  0x1000, 0x1001, 0x1004, 0x1005, 0x1010, 0x1011, 0x1014, 0x1015,
  0x1040, 0x1041, 0x1044, 0x1045, 0x1050, 0x1051, 0x1054, 0x1055,
  0x1100, 0x1101, 0x1104, 0x1105, 0x1110, 0x1111, 0x1114, 0x1115,
  0x1140, 0x1141, 0x1144, 0x1145, 0x1150, 0x1151, 0x1154, 0x1155,
  0x1400, 0x1401, 0x1404, 0x1405, 0x1410, 0x1411, 0x1414, 0x1415,
  0x1440, 0x1441, 0x1444, 0x1445, 0x1450, 0x1451, 0x1454, 0x1455,
  0x1500, 0x1501, 0x1504, 0x1505, 0x1510, 0x1511, 0x1514, 0x1515,
  0x1540, 0x1541, 0x1544, 0x1545, 0x1550, 0x1551, 0x1554, 0x1555,
  0x4000, 0x4001, 0x4004, 0x4005, 0x4010, 0x4011, 0x4014, 0x4015,
  0x4040, 0x4041, 0x4044, 0x4045, 0x4050, 0x4051, 0x4054, 0x4055,
  0x4100, 0x4101, 0x4104, 0x4105, 0x4110, 0x4111, 0x4114, 0x4115,
  0x4140, 0x4141, 0x4144, 0x4145, 0x4150, 0x4151, 0x4154, 0x4155,
  0x4400, 0x4401, 0x4404, 0x4405, 0x4410, 0x4411, 0x4414, 0x4415,
  0x4440, 0x4441, 0x4444, 0x4445, 0x4450, 0x4451, 0x4454, 0x4455,
  0x4500, 0x4501, 0x4504, 0x4505, 0x4510, 0x4511, 0x4514, 0x4515,
  0x4540, 0x4541, 0x4544, 0x4545, 0x4550, 0x4551, 0x4554, 0x4555,
  0x5000, 0x5001, 0x5004, 0x5005, 0x5010, 0x5011, 0x5014, 0x5015,
  0x5040, 0x5041, 0x5044, 0x5045, 0x5050, 0x5051, 0x5054, 0x5055,
  0x5100, 0x5101, 0x5104, 0x5105, 0x5110, 0x5111, 0x5114, 0x5115,
  0x5140, 0x5141, 0x5144, 0x5145, 0x5150, 0x5151, 0x5154, 0x5155,
  0x5400, 0x5401, 0x5404, 0x5405, 0x5410, 0x5411, 0x5414, 0x5415,
  0x5440, 0x5441, 0x5444, 0x5445, 0x5450, 0x5451, 0x5454, 0x5455,
  0x5500, 0x5501, 0x5504, 0x5505, 0x5510, 0x5511, 0x5514, 0x5515,
  0x5540, 0x5541, 0x5544, 0x5545, 0x5550, 0x5551, 0x5554, 0x5555
};
unsigned int morton_table(unsigned short x, unsigned short y) {
  return (unsigned int)MortonTable256[y >> 8]   << 17 |
         (unsigned int)MortonTable256[x >> 8]   << 16 |
         (unsigned int)MortonTable256[y & 0xFF] <<  1 |
         (unsigned int)MortonTable256[x & 0xFF];
}
```

### Bytes, with 64-bit multiplies

For x and y below 256. Each byte is spread out with one multiply, masked to one
bit per byte, and gathered into place with a second multiply. That's 11
operations. (Holger Bettag came up with it.)

```c
unsigned short morton_bytes_mul(unsigned char x, unsigned char y) {
  return ((x * 0x0101010101010101ULL & 0x8040201008040201ULL) *
           0x0102040810204081ULL >> 49) & 0x5555 |
         ((y * 0x0101010101010101ULL & 0x8040201008040201ULL) *
           0x0102040810204081ULL >> 48) & 0xAAAA;
}
```

### Binary magic numbers

Spread x and y (both below 65536) by shifting and masking with finer and finer
masks, then combine them.

```c
unsigned int morton_magic(unsigned int x, unsigned int y) {
  static const unsigned int B[] = {0x55555555, 0x33333333, 0x0F0F0F0F, 0x00FF00FF};
  static const unsigned int S[] = {1, 2, 4, 8};
  x = (x | (x << S[3])) & B[3];
  x = (x | (x << S[2])) & B[2];
  x = (x | (x << S[1])) & B[1];
  x = (x | (x << S[0])) & B[0];
  y = (y | (y << S[3])) & B[3];
  y = (y | (y << S[2])) & B[2];
  y = (y | (y << S[1])) & B[1];
  y = (y | (y << S[0])) & B[0];
  return x | (y << 1);
}
```

## Bytes in a word

### Does a word contain a zero byte?

`has_zero_byte` clears each byte's top bit and adds 0x7F, which carries into
the top bit unless the rest of the byte is zero. ORing the original top bits
back in leaves a top bit clear only where the whole byte was zero. The pretest
is cheaper, but it also fires when the top byte is 0x80. I checked all 2^32
words: those are its only false positives. `haszero` is the fastest
(Mycroft, 1987). Its yes/no answer is exact, but the per-byte flags can be
wrong above a real zero byte.

Fixed (bug 12): the original writes `haszero` with fixed 32-bit constants but
builds `hasvalue` on top of it with `~0UL/255`. On 64-bit Linux and macOS,
where `unsigned long` is 64 bits, `hasvalue` then only checks the low four
bytes. Here `haszero` uses the same width-independent constants as the other
macros. It's the same as `hasless(v, 1)`.

```c
bool has_zero_byte(unsigned int v) { return ~((((v & 0x7F7F7F7F) + 0x7F7F7F7F) | v) | 0x7F7F7F7F); }
bool has_no_zero_byte(unsigned int v) { return (v & 0xff) && (v & 0xff00) && (v & 0xff0000) && (v & 0xff000000); }
bool has_no_zero_byte_ptr(unsigned int v) {
  unsigned char * p = (unsigned char *) &v;
  return *p && *(p + 1) && *(p + 2) && *(p + 3);
}
bool has_zero_byte_pretested(unsigned int v) {
  bool hasZeroByte = ((v + 0x7efefeff) ^ ~v) & 0x81010100;
  if (hasZeroByte) {
    hasZeroByte = ~((((v & 0x7F7F7F7F) + 0x7F7F7F7F) | v) | 0x7F7F7F7F);
  }
  return hasZeroByte;
}
#define haszero(v) (((v) - ~0UL/255) & ~(v) & ~0UL/255*128)
```

### Has a byte equal to n, less than n, greater than n, or between m and n

These work on `unsigned long` words of any size. `hasvalue` XORs every byte
with n, so a match turns into a zero byte. `hasless` (n up to 128) and
`hasmore` (n up to 127) compare all bytes with n at once (Juha Järvi). The
`count` versions return how many bytes match. `likelyhasbetween` tests
m < byte < n (m up to 127, n up to 128) but can also report bytes equal to
n, so use it as a quick pretest. `hasbetween` gets it exactly right.

```c
#define hasvalue(x,n) \
(haszero((x) ^ (~0UL/255 * (n))))
#define hasless(x,n) (((x)-~0UL/255*(n))&~(x)&~0UL/255*128)
#define countless(x,n) \
(((~0UL/255*(127+(n))-((x)&~0UL/255*127))&~(x)&~0UL/255*128)/128%255)
#define hasmore(x,n) (((x)+~0UL/255*(127-(n))|(x))&~0UL/255*128)
#define countmore(x,n) \
(((((x)&~0UL/255*127)+~0UL/255*(127-(n))|(x))&~0UL/255*128)/128%255)
#define likelyhasbetween(x,m,n) \
((((x)-~0UL/255*(n))&~(x)&((x)&~0UL/255*127)+~0UL/255*(127-(m)))&~0UL/255*128)
#define hasbetween(x,m,n) \
((~0UL/255*(127+(n))-((x)&~0UL/255*127)&~(x)&((x)&~0UL/255*127)+~0UL/255*(127-(m)))&~0UL/255*128)
#define countbetween(x,m,n) (hasbetween(x,m,n)/128%255)
```

## Next bit permutation

Given a number with k bits set, this gives the next larger number with k bits
set: 00010011, 00010101, 00010110, 00011001 and so on. `t` fills in v's
trailing zeros. The lowest zero bit of `t` is the one that moves up, and the
ones below it get packed back at the bottom. The second version avoids `ctz`
but divides instead. v must be non-zero, and the next permutation has to fit
in 32 bits.

```c
unsigned int next_bit_permutation(unsigned int v) {
  unsigned int t = v | (v - 1);
  return (t + 1) | (((~t & -~t) - 1) >> (__builtin_ctz(v) + 1));
}
unsigned int next_bit_permutation_div(unsigned int v) {
  unsigned int t = (v | (v - 1)) + 1;
  return t | ((((t & -t) / (v & -v)) >> 1) - 1);
}
```

## Credits

The collection is by Sean Eron Anderson
(https://graphics.stanford.edu/~seander/bithacks.html), with contributions
from many people credited on that page. The people named above are the ones
the original credits for each technique.
