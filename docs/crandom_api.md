# Module crand: Pseudo Random Number Generators

This describes the API of module **crand**. It contains *pcg32* created by Melissa O'Neill, and an
*64-bit PRNG* created by Tyge Løvset. The PRNG can generate uniform and normal distributions.

## Types

| Name                  | Type definition                             | Used to represent...                 |
|:----------------------|:--------------------------------------------|:-------------------------------------|
| `crand_rng32_t`       | `struct {uint64_t state[2];}`               | The crandom type                     |
| `crand_uniform_i32_t` | `struct {int32_t offset; uint32_t range;}`  | The crandom element type             |
| `crand_uniform_f32_t` | `struct {float offset, range;}`             | crandom iterator                     |
| `crand_rng64_t`       | `struct {uint64_t state[4];}`               |                                      |
| `crand_uniform_i64_t` | `struct {int64_t offset; uint64_t range;}`  |                                      |
| `crand_uniform_f64_t` | `struct {double offset, range;}`            |                                      |
| `crand_normal_f64_t`  | `struct {double mean, stddev, ...;}`        |                                      |

## Header file

All cstr definitions and prototypes may be included in your C source file by including a single header file.
```c
#include "stc/crandom.h"
```

## Methods

```c
 1)     crand_rng32_t           crand_rng32_init(uint64_t seed);
 2)     crand_rng32_t           crand_rng32_with_seq(uint64_t seed, uint64_t seq);
 3)     uint32_t                crand_i32(crand_rng32_t* rng);
 4)     float                   crand_f32(crand_rng32_t* rng);
 5)     crand_uniform_i32_t     crand_uniform_i32_init(int32_t low, int32_t high);
 6)     int32_t                 crand_uniform_i32(crand_rng32_t* rng, crand_uniform_i32_t* dist);
 7)     uint32_t                crand_unbiased_i32(crand_rng32_t* rng, crand_uniform_i32_t* dist);
 8)     crand_uniform_f32_t     crand_uniform_f32_init(float low, float high); /*  */
 9)     float                   crand_uniform_f32(crand_rng32_t* rng, crand_uniform_f32_t* dist);
```
`1-2)` PRNG 32-bit engine initializers. `3)` Integer RNG with range \[0, 2^32). `4)` Float RNG with range \[0, 1). 
`5-6)` Uniform integer RNG with range \[*low*, *high*]. `7)` Unbiased version, see https://github.com/lemire/fastrange.
`8-9)` Uniform float RNG with range \[*low*, *high*).
```c
 1)     crand_rng64_t           crand_rng64_with_seq(uint64_t seed, uint64_t seq);
 2)     crand_rng64_t           crand_rng64_init(uint64_t seed);
 3)     uint64_t                crand_i64(crand_rng64_t* rng);
 4)     double                  crand_f64(crand_rng64_t* rng);
 5)     crand_uniform_i64_t     crand_uniform_i64_init(int64_t low, int64_t high);
 6)     int64_t                 crand_uniform_i64(crand_rng64_t* rng, crand_uniform_i64_t* dist);
 7)     crand_uniform_f64_t     crand_uniform_f64_init(double low, double high);
 8)     double                  crand_uniform_f64(crand_rng64_t* rng, crand_uniform_f64_t* dist);
 9)     crand_normal_f64_t      crand_normal_f64_init(double mean, double stddev);
10)     double                  crand_normal_f64(crand_rng64_t* rng, crand_normal_f64_t* dist);
```
`1-2)` PRNG 64-bit engine initializers. `3)` Integer generator, range \[0, 2^64). `4)` Double RNG with range \[0, 1).
`5-6)` Uniform integer RNG with range \[*low*, *high*]. `7-8)` Uniform double RNG with range \[*low*, *high*).
`9-10)` Normal-distributed double RNG were 99.7% of the values are within the range [*mean*-*stddev\*3*, *mean*+*stddev\*3*].

The method `crand_i64(crand_rng64_t* rng)` is an extremely fast PRNG suited for parallel usage, featuring
a Weyl-sequence as part of the state. It is faster than *sfc64*, *wyhash64*, *pcg*, and the *xoroshiro*
families of RPNGs. It does not require fast multiplication or 128-bit integer operations. The state is
256-bits, but updates only 192 bit per generated number. It can create 2^63 unique threads with minimum period
length of 2^64 per thread (expected period lengths 2^127). There is no *jump function*, but incrementing
the Weyl-increment by 2, achieves the same goal. Passes *PractRand*, tested up to 8TB output.

## Example
```c
#include <stdio.h>
#include <time.h>
#include <math.h>
#include "stc/crandom.h"
#include "stc/cstr.h"
#include "stc/cmap.h"
#include "stc/cvec.h"

// Declare unordered map: int -> int with typetag 'i'.
using_cmap(i, int, size_t);

// Comparison of map keys.
static int compare(cmap_i_entry_t *a, cmap_i_entry_t *b) {
    return c_default_compare(&a->first, &b->first);
}
// Declare vector of map entries, with comparison function.
using_cvec(e, cmap_i_entry_t, c_default_del, compare);


int main()
{
    enum {N = 10000000};
    const double Mean = -12.0, StdDev = 6.0, Scale = 74;

    printf("Demo of gaussian / normal distribution of %d random samples\n", N);

    // Setup random engine with normal distribution.
    uint64_t seed = time(NULL);
    crand_rng64_t rng = crand_rng64_init(seed);
    crand_normal_f64_t dist = crand_normal_f64_init(Mean, StdDev);

    // Create histogram map
    cmap_i mhist = cmap_i_init();
    for (size_t i = 0; i < N; ++i) {
        int index = (int) round( crand_normal_f64(&rng, &dist) );
        cmap_i_emplace(&mhist, index, 0).first->second += 1;
    }

    // Transfer map to vec and sort it by map entry keys.
    cvec_e vhist = cvec_e_init();
    c_foreach (i, cmap_i, mhist)
        cvec_e_push_back(&vhist, *i.val);
    cvec_e_sort(&vhist);

    // Print the gaussian bar chart
    cstr_t bar = cstr_init();
    c_foreach (i, cvec_e, vhist) {
        size_t n = (size_t) (i.val->second * StdDev * Scale * 2.5 / N);
        if (n > 0) {
            cstr_resize(&bar, n, '*');
            printf("%4d %s\n", i.val->first, bar.str);
        }
    }

    // Cleanup
    cstr_del(&bar);
    cmap_i_del(&mhist);
    cvec_e_del(&vhist);
}
```
Output:
```
Demo of gaussian / normal distribution of 10000000 random samples
 -29 *
 -28 **
 -27 ***
 -26 ****
 -25 *******
 -24 *********
 -23 *************
 -22 ******************
 -21 ***********************
 -20 ******************************
 -19 *************************************
 -18 ********************************************
 -17 ****************************************************
 -16 ***********************************************************
 -15 *****************************************************************
 -14 *********************************************************************
 -13 ************************************************************************
 -12 *************************************************************************
 -11 ************************************************************************
 -10 *********************************************************************
  -9 *****************************************************************
  -8 ***********************************************************
  -7 ****************************************************
  -6 ********************************************
  -5 *************************************
  -4 ******************************
  -3 ***********************
  -2 ******************
  -1 *************
   0 *********
   1 *******
   2 ****
   3 ***
   4 **
   5 *
```
