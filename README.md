# numeric

Number theory, rounding with explicit modes, interpolation, primes, combinatorics and descriptive statistics for SSPUR. It builds on the builtins (`Int.gcd`, `checked_*`, `clamp`, the `F64` cmath methods, `BigInt`) and adds what they leave out, in the spirit of Python's `math` and `statistics` modules, Apache Commons Math's `ArithmeticUtils`/`CombinatoricsUtils`/`DescriptiveStatistics` and Guava's `IntMath`/`LongMath`. Every function is pure, works for every Int including `int_min()` and `int_max()` without overflow traps unless documented, and is tested natively and in the interpreter.

Conventions: a function that can have no answer returns `Opt` (an empty list, a constant series, an Int overflow); an argument that can only be a caller's bug is a `where` refinement or `pre` clause. Statistics take `List[F64]` (`floats(xs)` converts Ints). A NaN anywhere in the input makes a numeric result NaN. SSPUR compares F64 with a total order (NaN equals itself and sorts last, `-0.0 < 0.0`), so the functions test for NaN and negative zero explicitly rather than relying on `==` and `<`.

## Example

```
use numeric.{HalfEven, HalfAwayFromZero, Exclusive, round_to, div_round, prime_factors, comb, mean, stdev, quantiles, linear_regression}

fn main() -> Unit ! log
= do
  log(round_to(2.675, 2, HalfAwayFromZero).str)
  log(div_round(7, 2, HalfEven).str)
  log(prime_factors(9223372036854775807).str)
  log(comb(100, 50).str)
  xs = [2.75, 1.75, 1.25, 0.25, 0.5, 1.25, 3.5]
  log("{mean(xs)} {stdev(xs)}")
  log(quantiles([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0], 4, Exclusive).str)
  log(linear_regression([1.0, 2.0, 3.0], [2.0, 4.0, 6.0]).str)
```

prints, natively and with `--interp`:

```
2.68
4
[7, 7, 73, 127, 337, 92737, 649657]
100891344545564193334812497256
some(1.6071428571428572) some(1.1713342006121947)
some([2.75, 5.5, 8.25])
some(Fit{slope: 2.0, intercept: 0.0, r_squared: 1.0})
```

## API

### Integers

| Definition | What it does |
|---|---|
| `int_max()`, `int_min()` | 2^63 - 1 and -2^63 |
| `checked_abs(x)` | `|x|`, or `none` for `int_min()` |
| `sign_int(x)`, `sign_f64(x)` | -1, 0 or 1; the F64 one keeps -0.0 and NaN |
| `gcd_all(xs)` | gcd of a list, never negative, 0 for `[]` |
| `checked_lcm(a, b)`, `lcm_all(xs)` | lcm, `none` on overflow; `lcm_all([]) == some(1)` |
| `egcd(a, b)` | `(g, x, y)` with `a * x + b * y == g` |
| `is_coprime(a, b)` | gcd is 1 |
| `floor_div`, `floor_mod` | Python's `//` and `%` (remainder takes the divisor's sign) |
| `ceil_div` | division rounded toward +infinity |
| `euclid_div`, `euclid_mod` | Euclidean division: the remainder is never negative |
| `addmod`, `mulmod` | `(a + b) mod m`, `(a * b) mod m` without overflow for any Ints |
| `mod_pow(b, e, m)`, `mod_inv(a, m)` | modular power and inverse (`none` when not coprime) |
| `isqrt`, `icbrt`, `iroot(n, k)` | exact floor of square, cube and k-th roots |
| `checked_pow(b, e)` | `b ^ e` or `none` on overflow (`big(b).pow(e)` for exact big powers) |
| `is_perfect_square`, `perfect_power`, `is_perfect_power` | square test; largest `(base, k)` with `base ^ k == n` |
| `is_power_of_two`, `next_power_of_two` | powers of two, `none` above 2^62 |

### Digits and radix

| Definition | What it does |
|---|---|
| `digits(n, base)`, `from_digits(ds, base)` | digits of `|n|`, most significant first, and back (`none` on a bad digit or overflow) |
| `digit_sum(n)`, `num_digits(n, base)`, `digital_root(n)` | decimal digit sum, digit count, repeated digit sum |
| `to_radix(n, base)`, `from_radix(s, base)` | bases 2 to 36, lowercase out, either case in, optional sign, `none` on anything else |

### Rounding

`Rounding` is `HalfUp` (ties toward +infinity), `HalfDown` (ties toward -infinity), `HalfEven` (banker's), `HalfAwayFromZero` (school rounding, Java's HALF_UP), `HalfTowardZero` (Java's HALF_DOWN), `Up` (away from zero), `Down` (toward zero), `Ceiling` and `Floor`.

| Definition | What it does |
|---|---|
| `div_round(a, b, mode)` | `a / b` rounded with `mode`, exact |
| `round_int_to_multiple(n, step, mode)` | nearest multiple by `mode`, `none` on overflow |
| `round_to(x, places, mode)` | F64 to `places` decimals (negative: tens, hundreds, ...) |
| `round_to_multiple(x, step, mode)` | F64 to a multiple of `step` |

F64 rounding is exact on the shortest decimal that prints as the value (its `.str`), then converted back to the nearest double. So `round_to(2.675, 2, HalfAwayFromZero) == 2.68` and `round_to_multiple(0.3, 0.1, Floor) == 0.3`, as a person reading the numbers expects, where rounding the binary values gives 2.67 (Python's `round`) and 0.2 (`(0.3 / 0.1).floor`). The result is still a double: `round_to(x, 2, m)` is the double nearest the two-place decimal, not that decimal. Use `Dec` for money.

### Interpolation, closeness and angles

| Definition | What it does |
|---|---|
| `lerp(a, b, t)`, `inverse_lerp(a, b, v)`, `remap(v, in_lo, in_hi, out_lo, out_hi)` | linear interpolation and its inverse (`none` for an empty range) |
| `smoothstep(e0, e1, x)` | GLSL smoothstep, clamped |
| `approx_equal(a, b, rel_tol, abs_tol)`, `is_close(a, b)` | Python's `math.isclose` |
| `deg_to_rad`, `rad_to_deg` | conversions |
| `wrap(x, lo, hi)`, `normalize_degrees`, `normalize_radians` | wrap into `lo .. hi`, `0 .. 360`, `-pi .. pi` |

### Primes

| Definition | What it does |
|---|---|
| `is_prime(n)` | deterministic for every Int (Miller-Rabin, 12 bases) |
| `primes_up_to(limit)` | sieve of Eratosthenes, limit up to 10,000,000 |
| `next_prime(n)`, `prev_prime(n)` | `none` past the largest Int prime or below 2 |
| `prime_factors(n)`, `factorize(n)` | with multiplicity, or as `(prime, exponent)`; trial division then Pollard's rho, any Int |
| `divisors(n)`, `totient(n)` | sorted divisors, Euler's phi |

### Combinatorics

| Definition | What it does |
|---|---|
| `factorial(n)`, `factorial_int(n)` | exact BigInt (n up to 100,000), or Int up to 20! |
| `comb(n, k)`, `comb_int(n, k)` | binomial coefficient; 0 outside `0..=n`; Int version `none` on overflow |
| `perm(n, k)`, `perm_int(n, k)` | `n! / (n - k)!` |
| `multinomial(ks)`, `catalan(n)` | multinomial coefficient, Catalan numbers |
| `fibonacci(n)`, `fibonacci_int(n)` | fast doubling, BigInt or Int up to F(92) |

### Statistics

| Definition | What it does |
|---|---|
| `floats(xs)` | `List[Int]` to `List[F64]` |
| `checked_sum`, `checked_product` | Int sum and product, `none` on overflow |
| `compensated_sum`, `cumulative_sum`, `product` | Neumaier summation, running sums, F64 product |
| `mean`, `weighted_mean`, `geometric_mean`, `harmonic_mean` | averages; `mean` survives overflow of the sum |
| `median`, `median_low`, `median_high` | as Python's `statistics` |
| `mode`, `multimode` | for any type, first seen on ties |
| `pvariance`, `variance`, `pstdev`, `stdev` | population and sample, Welford's algorithm |
| `Quantile`, `quantile(xs, p, method)`, `percentile(xs, p)`, `quantiles(xs, n, method)` | `Linear` (R-7, NumPy's default), `Exclusive` (R-6, Python's default), `Lower`, `Higher`, `Midpoint`, `Nearest`, `NearestRank` |
| `iqr`, `value_range` | Q3 - Q1, max - min |
| `zscores`, `min_max_normalize` | standardize, or scale to `0 .. 1` (`none` for constant data) |
| `covariance`, `correlation`, `ranks`, `spearman` | sample covariance, Pearson, average ranks, Spearman |
| `Fit`, `linear_regression(xs, ys)` | least squares slope, intercept and r^2 |
| `Bin`, `histogram(xs, bins)`, `histogram_range(xs, lo, hi, bins)` | equal-width bins |
| `moving_average(xs, w)`, `exponential_moving_average(xs, alpha)` | trailing simple and exponential averages |

## Out of scope

- Arbitrary-precision decimals and exact money arithmetic: use the builtin `Dec` and `Ratio`. Correctly rounded `fsum` (Python's): `compensated_sum` is Neumaier's, accurate to about one ulp, not always the nearest double.
- Linear algebra, matrices, complex analysis (the builtin `Complex` covers complex numbers), special functions beyond the builtin `gamma`/`erf`, numerical integration, root finding and optimization (Commons Math's `analysis` and `optim` packages).
- Probability distributions and random sampling (the builtins have seeded generators and distributions), hypothesis tests and confidence intervals.
- Multiple regression and other models; streaming or windowed statistics objects.

## Install

```
sspur add numeric
```

MIT licensed.
