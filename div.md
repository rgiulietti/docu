# Signed truncating integer division by non-powers of $2$ constants

Raffaello Giulietti, v2023-10-29-00

In terms of running time, division is the most expensive of the integer arithmetical/logical operations on contemporary CPUs.

Given an integer compile-time constant `d` (whether during static compilation, or during JIT runtime compilation), the goal is to replace later computations of `x / d` with faster code producing the same outcome, where integer `x` is _not_ known at compile-time.
A preparatory work, done at compile-time and that depends on `d` alone, leads to faster code for the division later on at run-time.

Detailed proofs of the core algorithm presented here can be found [elsewhere](https://drive.google.com/file/d/1gp5xv4CAa78SVgCeWfGqqI4FfYYYuNFb), in §10.1.

## Notation

* We write $-2^k$ for $-(2^k)$, although the former really means $(-2)^k$ (negation binds tighter than binary operations like exponentiation).
* $W$ denotes the word size (e.g., $32$ or $64$).
* $/$ denotes usual division over the real numbers (when not appearing in code as `/`).
* $\div$ denotes truncating division over the real numbers: $x \div d = \lfloor x / d\rfloor$ if $x / d \ge 0$, $x \div d = \lceil x / d\rceil$ otherwise, where $d \ne 0$.
In Java code, with `int x, d`, and absent overflows, `x / y`${}= x \div d$.

## Positive divisor: $d > 0$

Let integer $d$ meet both $0 < d < 2^{W-1}$.
Compute

$$m = (W - 1) + \lceil \log_2 d\rceil, \quad c = \lceil 2^m / d\rceil$$

Then, for integer $x$ meeting $0 \le x < 2^{W-1}$ we have

$$x \div d = \lfloor x c / 2^m\rfloor$$

and for integer $x$ meeting $-2^{W-1} \le x < 0$ we have

$$x \div d = \lfloor (x c - 1) / 2^m\rfloor + 1$$

All computations can be done in signed $W$-bit arithmetic (for $m$), or signed $2 W$-bit arithmetic (for other values), and the quotient fits in $W$ bits.

## Negative divisor: $d < 0$

For integer $d$ meeting $-2^{W-1} \le d < 0$ this case reduces to the above case by using the identity

$x \div d = -(x \div (-d))$

## Truncated remainder

The _truncated remainder_ $r$ is defined as

$$r = x - (x \div d) d$$

and can be computed using $W$-bit arithmetic once $x \div d$, a $W$-bit value, is known.

## Java code for the `int` case ($W = 32 = {}$`Integer.SIZE`)

### Divisor $d > 0$

We add the restriction that $d \ne 2^k$ for all integers $k \ge 0$.
That is, we exclude powers of $2$, including $d = 1$, as divisors.
The reasoon is that there are faster computations when $d$ is a power of $2$, and that it simplifies the code a bit.
This implies
$$W < m \le 2 W - 2$$

#### Preliminary check to exclude powers of $2$ divisors

To check whether an _unsigned_ `int d` (an `int` interpreted as an unsigned value) is a power of $2$:

```java
(d & (d - 1)) == 0;     // holds iff d is a power of 2, or if d = 0
```

#### Compile-time preparation

```java
int m = (2 * Integer.SIZE - 1) - Integer.numberOfLeadingZeros(d);
long c = (1L << m) / d + 1;     // the division only happens at compile-time
```

No overflows occur.

#### Truncating division

```java
long p = x * c;     // to reduce latency, schedule the product before computing s
int s = x >>> (Integer.SIZE - 1);   // 0 if x >= 0; 1 if x < 0
int q = (int) (p - s >> m) + s;
```

again without overflows.

If accessing the high `int` half of a `long`, and shifting it, is faster than just shifting a `long`, the line for `q` can be replaced by

```java
int q = (high_half(p - s) >> (m - Integer.SIZE)) + s;
```

where the shift distance `m - Integer.SIZE` is a compile-time constant.

Of course, in compile-time contexts where `x` is known to be non-negative, the division can be simplified to the one-liner

```java
int q = (int) (x * c >> m);
```

resp.

```java
int q = high_half(x * c) >> (m - Integer.SIZE);
```

### Divisor $d < 0$

Again, we impose the restriction $-d \ne 2^k$ for all integers $k \ge 0$ (which, in fact, excludes the extremes $d = -1$ and $d = -2^{W-1}$).
Computing `-d` overflows to itself when `d = Integer.MIN_VALUE`, but this is not an issue: indeed, when seen as an unsigned value, this is a power of $2$, and thus excluded during the preliminary check.

#### Preparation

```java
int m = (2 * Integer.SIZE - 1) - Integer.numberOfLeadingZeros(-d);
long c = (-1L << m) / d + 1;    // the division only happens at compile-time
```

#### Division

```java
long p = x * c;     // to reduce latency, schedule the product before computing s
int s = x >>> (Integer.SIZE - 1);   // 0 if x >= 0; 1 if x < 0
int q = -((int) (p - s >> m) + s);
```

with similar variations as above.