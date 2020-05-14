
[![Build Status](https://travis-ci.org/Quid2/flat.svg?branch=master)](https://travis-ci.org/Quid2/flat)
[![Hackage version](https://img.shields.io/hackage/v/flat.svg)](http://hackage.haskell.org/package/flat)

Haskell implementation of [Flat](http://quid2.org/docs/Flat.pdf), a principled, portable and efficient binary data format ([specs](http://quid2.org)).

### How To Use It For Fun and Profit

To (de)serialise a data type, make it an instance of the `Flat` class.

There is `Generics` based support to automatically derive instances of additional types.

Let's see some code, we need a couple of extensions:

```haskell
{-# LANGUAGE DeriveGeneric, DeriveAnyClass #-}
```

Import the Flat library:

```haskell
import Flat
```

Define a couple of custom data types, deriving Generic and Flat:

```haskell
data Direction = North | South | Center | East | West deriving (Show,Generic,Flat)
```

```haskell
data List a = Nil | Cons a (List a) deriving (Show,Generic,Flat)
```

For encoding, use `flat`, for decoding, use `unflat`:

```haskell
unflat . flat $ Cons North (Cons South Nil) :: Decoded (List Direction)
-> Right (Cons North (Cons South Nil))
```


For the decoding to work correctly, you will naturally need to know the type of the serialised data. This is ok for applications that do not require long-term storage and that do not need to communicate across independently evolving agents. For those who do, you will need to supplement `flat` with something like [zm](https://github.com/Quid2/zm).

#### Define Instances for Abstract/Primitive types

 A set of primitives are available to define `Flat` instances for abstract or primitive types.

 Instances for some common, primitive or abstract data types (Bool,Words,Int,String,Text,ByteStrings,Tuples, Lists, Sequences, Maps ..) are already defined in [Data.Flat.Instances](https://github.com/Quid2/flat/blob/master/src/Data/Flat/Instances.hs).

#### Optimal Bit-Encoding

A pecularity of Flat is that it uses an optimal bit-encoding rather than the usual byte-oriented one.

 To see this, let's define a pretty printing function: `bits` encodes a value as a sequence of bits, `prettyShow` displays it nicely:

```haskell
p :: Flat a => a -> String
p = prettyShow . bits
```

Now some encodings:

```haskell
p West
-> "111"
```


```haskell
p (Nil::List Direction)
-> "0"
```


```haskell
aList = Cons North (Cons South (Cons Center (Cons East (Cons West Nil))))
p aList
-> "10010111 01110111 10"
```


As you can see, `aList` fits in less than 3 bytes rather than 11 as would be the case with other Haskell byte oriented serialisation packages like `binary` or `store`.

For the serialisation to work with byte-oriented devices or storage, we need to add some padding:

```haskell
f :: Flat a => a -> String
f = prettyShow . paddedBits
```

```haskell
f West
-> "11100001"
```


```haskell
f (Nil::List Direction)
-> "00000001"
```


```haskell
f $ Cons North (Cons South (Cons Center (Cons East (Cons West Nil))))
-> "10010111 01110111 10000001"
```

The padding is a sequence of 0s terminated by a 1 running till the next byte boundary (if we are already at a byte boundary it will add an additional byte of value 1, that's unfortunate but there is a good reason for this, check the [specs](http://quid2.org/docs/Flat.pdf)).

Byte-padding is automatically added by the function `flat` and removed by `unflat`.

### Performance

For some hard data, see this [comparison of the major haskell serialisation libraries](https://github.com/haskell-perf/serialization).

Briefly:
 * Size: `flat` produces significantly smaller binaries than all other libraries (3/4 times usually)
 * Serialization: `store`, `persist` and `flat` are faster
 * Deserialization: `store`, `flat`, `persist` and `cereal` are faster
 * Transfer time (serialisation time + transport time on the network + deserialisation at the receiving end): `flat` is usually faster for all but the highest network speeds

### Compatibility

#### [GHC](https://www.haskell.org/ghc/) 

Tested with:
  * [ghc](https://www.haskell.org/ghc/) 7.10.3, 8.0.2, 8.2.2, 8.4.4 and 8.6.3 (x64)

Should also work with (not recently tested):
  * [ghc](https://www.haskell.org/ghc/) 7.10.3/LLVM 3.5.2 (Arm7)

####  [GHCJS](https://github.com/ghcjs/ghcjs)
 
Passes all tests in the `flat` testsuite, except for those relative to short bytestrings (Data.ByteString.Short) that are unsupported by `ghcjs`.

Check [stack-ghcjs.yaml](https://github.com/Quid2/flat/blob/master/stack-ghcjs.yaml) to see with what versions of `ghcjs` it has been tested.

If you use a different version of `ghcjs`, you might want to run the test suite by setting your compiler in [stack-ghcjs.yaml](https://github.com/Quid2/flat/blob/master/stack-ghcjs.yaml) and then running:

`stack test --stack-yaml=stack-ghcjs.yaml`

NOTE: Versions prior to 0.33 encode `Double` values incorrectly when they are not aligned with a byte boundary.

NOTE: A native [TypeScript/JavaScript version](https://github.com/Quid2/ts) of `flat` is under development.

#### [ETA](https://eta-lang.org/)

It builds (with etlas 1.5.0.0 and eta eta-0.8.6b2) but currently fails the test suite.

### Installation

Get the latest stable version from [hackage](https://hackage.haskell.org/package/flat).

### Known Bugs and Infelicities

#### Longish compilation times

'flat` relies more than other serialisation libraries on extensive inlining for its good performance, this unfortunately leads to longer compilation times. 

If you have many data types or very large ones this might become an issue.

A couple of good practices that will eliminate or mitigate this problem are:

* During development, turn optimisations off (`stack --fast` or `-O0` in the cabal file).

* Keep your serialisation code in a separate module(s).

#### Data types with more than 512 constructors are currently unsupported

See also the [full list of open issues](https://github.com/Quid2/flat/issues).

### Acknowledgements

 `flat` reuses ideas and readapts code from various packages, mainly: `store`, `binary-bits` and `binary` and includes contributions from Justus Sagemüller.
