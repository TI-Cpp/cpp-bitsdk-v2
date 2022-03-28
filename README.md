# bitsdk2: bitsdk improved

|Note|
|----|
|This version of bitsdk2 is for C++17.

bitsdk2::bitsdk2 is a header only library. It provides the same functionality as [std::bitsdk](http://en.cppreference.com/w/cpp/utility/bitsdk) with the
following extentions/changes.


Focus was set on having as many functions
implemented as [constexpr](http://en.cppreference.com/w/cpp/language/constexpr)
as possible. Moreover a second template parameter (with appropriate default)
allows control of the underlying data structure (see below).
* Copy and move constructors are specified constexpr.
* Additional constexpr constructor `bitsdk2( std::array<T,N> const & )`, where `T` needs not necessarily be equal to `base_t`. `T` has to be an unsigned integral type.
* Conversion from and to `std::bitsdk`.
* Operators implemented as constexpr are `~`, `==`, `!=`, `|`, `&`, `^`, `<<` (shift left), `>>` (shift right), `[]` (bit access).
* Non-const operators implemented as constexpr are `<<=`, `>>=`, `|=`, `&=`, `^=`
* Functions implemented as constexpr are `test`, `none`, `any`, `all`, `count`, `to_ulong`, `to_ullong`.
* Non-const functions implemented as constexpr are `set`, `reset`, `flip`.
* Additional constexpr operator `+` for adding two bitsdk2 objects.
* Additional constexpr operators `++`, `--`, `+=`.
* Additional constexpr operators `<`, `>`, `<=`, `>=`.
* Additional constexpr functions `rotate_left` and `rotate_right` for binary rotations.
* Additional constexpr member functions `rotate_left` and `rotate_right`.
* Additional member function `to_hex_string()` (see below).
* Additional constexpr member function `test_set( size_t bit, bool value= true )`, which sets or clears the specified bit and returns its previous state. Throws `out_of_range` if bit >= N.
* Additional constexpr function `difference`, which computes the set difference (`bs1 & ~bs2`) of two bitsdk2 objects.
* Additional constexpr member function `difference`.
* Additional constexpr member functions `find_first()` and `find_next(size_t)` returning the index of the  first (next) bit set. Returning npos if all (remaining) bits are false.
* Additional constexpr function `complement2(bs)` computing the [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) (~bs +1).
* Additional constexpr member function `complement2`.
* Additional constexpr function `reverse`, which returns argument with bits reversed.
* Additional constexpr member function `reverse`.
* Additional constexpr function `convert_to<n>` for converting an *m*-bit bitsdk2 into an *n*-bit bitsdk2.
* Additional constexpr function `convert_to<n,T>` for converting an *m*-bit bitsdk2 into an *n*-bit bitsdk2 with `base_t=T`.
* Constexpr member function `data()` gives read access to the underlying `array<base_t,N>`. Here element with index zero is the least significant word.
* Additional constexpr functions `zip_fold_and` and `zip_fold_or`. See below for details.

## Template parameters and underlying data type
`bitsdk2` is declared as
```.cpp
template< size_t N, class T >
class bitsdk2;
```
`N` is the number of bits and `T` has to be an unsigned
[integral type](http://en.cppreference.com/w/cpp/types/is_integral). Data
represented by `bitsdk2` objects are stored in elements of type
`std::array<T,n_array>`.

`T` defaults
to `uint8_t`, `uint16_t`, or `uint32_t` if `N` bits fit into these integers
(and the type is supported by the system).
`T` defaults to `unsigned long long` otherwise. The following aliases and
constants are public within `bitsdk2`:
* `using base_t= T;`
* `size_t n_array;` Number of words in underlying array
* `using array_t= std::array<T,n_array>;` Underlying data type

## to_hex_string
Function `to_hex_string` takes - as an optional argument - an element of type
`hex_params`, which is defined as
```.cpp
template< class CharT = char,
          class Traits = std::char_traits<CharT>,
          class Allocator = std::allocator<CharT> >
struct hex_params
{
  using str_t= std::basic_string<CharT,Traits,Allocator>;

  CharT        zeroCh=         CharT( '0' );
  CharT        aCh=            CharT( 'a' );
  bool         leadingZeroes=  true;
  bool         nonEmpty=       true;
  str_t        prefix;
};
```
It allows fine tuning the outcome of the function. In the following
examples the output is shown in the comments.
```.cpp
bitsdk2<16>  b16_a( "0000101000011111" );
bitsdk2<16>  b16_b;
std::cout
  << b16_a.to_hex_string() << '\n'                                    // 0a1f
  << b16_a.to_hex_string( hex_params<>{'0', 'A', false, true, "0x"})  // 0xA1F
  << '\n'
  << b16_b.to_hex_string() << '\n'                                    // 0000
  << b16_b.to_hex_string( hex_params<>{'0', 'a', false, false, "0X"}) // 0X
  << '\n';
```

## zip\_fold\_&ast;
Functions `zip_fold_and(bs1,bs2,f)` and `zip_fold_or(bs1,bs2,f)` expect two
variables of type `bitsdk2` and a functional object `f`.
The latter must accept two variables of type `base_t` and return a `bool`.
`zip_fold_*` are mapped over the underlying
`std::array<base_t,n>`s. They will
[short-circuit](http://en.cppreference.com/w/cpp/language/operator_logical)
if possible, which can result in performance advantages.
`zip_fold_and` returns `true` if `f`
returns `true` for each pair of `base_t`s, while `zip_fold_or` returns `true`
if `f` returns `true` for at least one pair of `base_t`s.
In other words `zip_fold_and` and `zip_fold_or` are similar to
[`std::inner_product(...,BinOp1 op1,BinOp2 op2)`](http://en.cppreference.com/w/cpp/algorithm/inner_product)
with `op1` set to `&&` and `||`, resp.

For instance `is_subset_of` as proposed in [p0125r0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0125r0.html)
can be implemented as follows.
```.cpp
template<size_t N,class T>
constexpr
bool
is_subset_of( bitsdk2::bitsdk2<N,T> const &bs1,
              bitsdk2::bitsdk2<N,T> const &bs2 ) noexcept
{
  using base_t= T;
  return bitsdk2::zip_fold_and( bs1, bs2,
                                []( base_t v1, base_t v2 ) noexcept
                                  // Any bit unset in v2 must not be set in v1
                                  { return (v1 & ~v2) == 0; } );
}

constexpr bitsdk2::bitsdk2<7>    b7_a( 0b1000101ull );
constexpr bitsdk2::bitsdk2<7>    b7_b( 0b1010101ull );
static_assert( is_subset_of( b7_a, b7_b), "" );
```

Similarly an `unequal` function can be defined as
```.cpp
template<size_t N,class T>
bool
unequal( bitsdk2::bitsdk2<N,T> const &bs1, bitsdk2::bitsdk2<N,T> const &bs2 )
{
  using base_t= T;
  return bitsdk2::zip_fold_or( bs1, bs2,
                               []( base_t v1, base_t v2 ) noexcept
                                 { return v1 != v2; } );
}
```

## Trivia
The following code shows a counter based on a 128-bit integer. If the
counter gets incremented once at each nanosecond, you have to wait for
overflow for *only* [1.078 * 10<sup>22</sup> years](http://www.wolframalpha.com/input/?i=2%5E128+nanoseconds).
```.cpp
bitsdk2::bitsdk2<128> c;
for( ;; ++c ) {}
```

## Caveats
* bitsdk2 requires a C++17 compliant compiler.
* Tested with gcc 7 and clang 5.
