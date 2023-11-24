# Haskell WATs

This is a collection of Haskell's WATs

See also [the list of dangerous functions](https://github.com/NorfairKing/haskell-dangerous-functions).

## Read instances for integral types

These instances use [`fromIntegral`, which is a dangerous function](https://github.com/NorfairKing/haskell-dangerous-functions#fromintegral-and-frominteger).

```
ghci> import Data.Word
ghci> import Data.Int
ghci> read "128" :: Int8
-128
ghci> read "256" :: Word8
0
ghci> read "60000" :: Int16
-5536
ghci> read "70000" :: Word16
4464
ghci> read "2147483649" :: Int32
-2147483647
ghci> read "42147483649" :: Word32
3492777985
ghci> read "5000000000000000000000" :: Int64
932356024711512064
ghci> read "18446744073709551617" :: Word
1
ghci> read "18446744073709551617" :: Int
1
ghci> read "28446744073709551617" :: Word
10000000000000000001
ghci> read "-1" :: Word8
255
ghci> read "-1" :: Word16
65535
ghci> read "-1" :: Word32
4294967295
ghci> read "-1" :: Word64
18446744073709551615
ghci> read "-1" :: Word
18446744073709551615
```

There's a [GHC issue about fixing this](https://gitlab.haskell.org/ghc/ghc/-/issues/24216).

## Eq Double

``` haskell
Prelude> let nan = read "NaN" :: Double
Prelude> nan == nan
False
Prelude> nan /= nan
True
```

You might think "That's just the way IEEE 754 floating point numbers work.", and I would agree with you if [Rust hadn't done it right](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html).

This problem has some interesting nasty side-effects:

* You can never use `Double` as the key in a Map:

```
Prelude> import qualified Data.Map as M
Prelude M> M.fromList [(nan, 1), (nan, 2)]
fromList [(NaN,1),(NaN,2)]
```

* You can never use `Double` as the key in a HashMap:

```
Prelude> import qualified Data.HashMap.Strict as HM
Prelude HM> HM.fromList [(nan, 1), (nan, 2)]
fromList [(NaN,1),(NaN,2)]
```

## Ord Double

``` haskell
Prelude> let nan = read "NaN" :: Double
Prelude> nan >= nan
False
Prelude> nan > nan
False
Prelude> nan <= nan
False
Prelude> nan < nan
False
Prelude> compare nan nan
GT
```

You might think "That's just the way IEEE 754 floating point numbers work.", and I would agree with you if [Rust hadn't done it right](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html).
In particular, `compare nan nan` being `GT` is certainly inconsistent.

## Real Double

``` Haskell
Prelude> let nan = read "NaN" :: Double
Prelude> toRational nan
(-269653970229347386159395778618353710042696546841345985910145121736599013708251444699062715983611304031680170819807090036488184653221624933739271145959211186566651840137298227914453329401869141179179624428127508653257226023513694322210869665811240855745025766026879447359920868907719574457253034494436336205824) % 1
Prelude> realToFrac nan -- With -O0
-Infinity
Prelude> realToFrac nan
NaN
```

## RealFrac Double

```
Prelude> let nan = read "NaN" :: Double
Prelude> properFraction nan
(-269653970229347386159395778618353710042696546841345985910145121736599013708251444699062715983611304031680170819807090036488184653221624933739271145959211186566651840137298227914453329401869141179179624428127508653257226023513694322210869665811240855745025766026879447359920868907719574457253034494436336205824,0.0)
truncate nan
-269653970229347386159395778618353710042696546841345985910145121736599013708251444699062715983611304031680170819807090036488184653221624933739271145959211186566651840137298227914453329401869141179179624428127508653257226023513694322210869665811240855745025766026879447359920868907719574457253034494436336205824
Prelude> round nan
-269653970229347386159395778618353710042696546841345985910145121736599013708251444699062715983611304031680170819807090036488184653221624933739271145959211186566651840137298227914453329401869141179179624428127508653257226023513694322210869665811240855745025766026879447359920868907719574457253034494436336205824
Prelude> floor nan
-269653970229347386159395778618353710042696546841345985910145121736599013708251444699062715983611304031680170819807090036488184653221624933739271145959211186566651840137298227914453329401869141179179624428127508653257226023513694322210869665811240855745025766026879447359920868907719574457253034494436336205824
Prelude> ceiling nan
-269653970229347386159395778618353710042696546841345985910145121736599013708251444699062715983611304031680170819807090036488184653221624933739271145959211186566651840137298227914453329401869141179179624428127508653257226023513694322210869665811240855745025766026879447359920868907719574457253034494436336205824
Prelude> properFraction nan :: (Int, Double)
(0,0.0)
Prelude> round nan :: Int
0
Prelude> floor nan :: Int
0
Prelude> ceiling nan :: Int
0
```
#### Rounding
```haskell
main = print (round (1e206 :: Double) :: Int)
```
Depending on the ghc optimization flag double rounding overflows differently:
```haskell
$ ghc round.hs && ./round
[1 of 1] Compiling Main             ( round.hs, round.o )
Linking round ...
0
$ ghc -O1 round.hs && ./round
[1 of 1] Compiling Main             ( round.hs, round.o ) [Optimisation flags changed]
Linking round ...
-9223372036854775808
```
This is because when optimization is turned on there are rewrite rules that use `double2Int` implemented in C. Therefore all of these potentially have a problem like that:
```
{-# RULES
"properFraction/Double->Integer"    properFraction = properFractionDoubleInteger
"truncate/Double->Integer"          truncate = truncateDoubleInteger
"floor/Double->Integer"             floor = floorDoubleInteger
"ceiling/Double->Integer"           ceiling = ceilingDoubleInteger
"round/Double->Integer"             round = roundDoubleInteger
"properFraction/Double->Int"        properFraction = properFractionDoubleInt
"truncate/Double->Int"              truncate = double2Int
"floor/Double->Int"                 floor = floorDoubleInt
"ceiling/Double->Int"               ceiling = ceilingDoubleInt
"round/Double->Int"                 round = roundDoubleInt
  #-}
```

## Num Int

```
Prelude> minBound * (-1) :: Int
-9223372036854775808
Prelude> abs (minBound :: Int)
-9223372036854775808
Prelude> minBound `div` (-1) :: Int
*** Exception: arithmetic overflow
Prelude> minBound `quot` (-1) :: Int
*** Exception: arithmetic overflow
```

Do we want modular arithmetic on `Int` or do we want to throw errors on (over|under)flow?

## Enum Rational and Enum Double

`Rational` is an `Enum`, which makes no sense because the `Rational` values are not enumerable in order.
On top of that:

```
Prelude> [1..2] :: [Rational]
[1 % 1,2 % 1]
Prelude> fromEnum (1 :: Rational)
1
Prelude> fromEnum (1.35 :: Rational)
1
```

`Double` can be an `Enum`, but it is implemented as a WAT:

```
Prelude> [1..2] :: [Double]
[1.0,2.0]
Prelude> fromEnum (1 :: Double)
1
Prelude> fromEnum (1.35 :: Double)
1
```

This gets programmers into problem because types like `Micro` _are_ implemented correctly:

```
Prelude> Import Data.Fixed
Prelude Data.Fixed> length ([1..2] :: [Micro])
1000001
```

## `Ratio` with fixed-size underlying types

(Recall (from the docs); "The numerator and denominator have no common factor and the denominator is positive.")

You can end up with invalid `Ratio` values using `Num` functions:

```
Prelude Data.Int Data.Ratio> let r = 1 % 12 :: Ratio Int8
Prelude Data.Int Data.Ratio> r - r
0 % (-1)
Prelude Data.Int Data.Ratio> r + r
3 % (-14)
> r * r
1 % (-112)
```

## Do block without a monad

This 'just works':

```
myTest :: Int
myTest = do
  let x = 5
  x
```

## Text round trip

`Text` values can only contain valid Unicode, so `Char` values from U+D800 to U+DFFF are replaced by U+FFFD, the replacement character.

``` hs
>>> any (\ char -> [char] == Text.unpack (Text.pack [char])) ['\xd800' .. '\xdfff']
False
```

This means that round-tripping a `String` through `Text` may not give you what you started with.

``` hs
>>> let string = "haskell \xd800 wat"
>>> string
"haskell \55296 wat"
>>> Text.pack string
"haskell \65533 wat"
```

## `Fixed` precision

[`Fixed`](https://hackage.haskell.org/package/base-4.15.0.0/docs/Data-Fixed.html) data types can silently lose precision from literal values.
For example the `Centi` type is supposed to have one decimal of precision.
Literals values are truncated without warning.

``` hs
>>> 1.21 :: Deci
1.2
>>> 1.29 :: Deci
1.2
```

The [`overflowed-literals`](https://downloads.haskell.org/~ghc/9.0.1/docs/html/users_guide/using-warnings.html#ghc-flag--Woverflowed-literals) warning catches this for integral values.
Unfortunately it doesn't work for fixed- or floating-point values.
<https://gitlab.haskell.org/ghc/ghc/-/issues/13232>

## Foldable tuples

The WAT here is not the behaviour per se (because you can figure that out from the kind of Foldable), but rather that someone thought this instance was a good idea.

```
Prelude> length ('a','b')
1
Prelude> maximum (2,1)
1
Prelude> minimum (1,2)
2
Prelude> sum (2,1)
1
Prelude> and (False, True)
True
Prelude> or (True, False)
False
```

## Foldable `Complex`

No one use these, as far as I can tell, so it doesn't really matter, but these are amazing.
Just so you know, `a :+ b` is the value that represents `a + ib`.

```
Prelude Data.Complex> length (1 :+ 1) -- 1 + 1i
2
Prelude Data.Complex> null (0 :+ 0) -- 0 + 0i
False
Prelude Data.Complex> sum (2 :+ 2) -- 2 + 2i
4
Prelude Data.Complex> product (3 :+ 3) -- 3 + 3i
9
```
