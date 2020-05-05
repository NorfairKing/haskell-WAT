# Haskell WATs

This is a collection of Haskell's WATs

## Eq Double

``` haskell
Prelude> let nan = read "NaN" :: Double
Prelude> nan == nan
False
Prelude> nan /= nan
True
```

You might think "That's just the way IEEE 753 floating point numbers work.", and I would agree with you if [Rust hadn't done it right](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html).

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

You might think "That's just the way IEEE 753 floating point numbers work.", and I would agree with you if [Rust hadn't done it right](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html).
In particular, `compare nan nan` being `False` is certainly inconsistent.

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


## Do block without a monad

This 'just works':

```
myTest :: Int
myTest = do
  let x = 5
  x
```
