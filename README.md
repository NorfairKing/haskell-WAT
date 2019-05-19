# Haskell WATs

This is a collection of Haskell's WATs


``` Haskell
> let nan = read "NaN" :: Double
> realToFrac nan -- With -O0
-Infinity
> realToFrac nan
NaN
```
