stdio: standard input and output
================================

This library is an effort trying to improve and standardize Haskell's IO interface with pack data types.

+ A faster/simpler vector type, which packed bytes type built on.
+ Binary and textual `Parser` and `Builder` for compact bytes.
+ A UTF-8 based text type for text process.
+ A simper `Handler` for IO, include file and network module.
+ A compact `FilePath` type.

This package is designed from ground, with two goals: simplicity and performance, join in!


Guide
-----

I will try to give a briefly introduction on this package to help you get started, the first step is to get some knowledge on [primitive](http://hackage.haskell.org/package/primitive), it's well documented and used widely as the interface to GHC's primitive operations, then we can get started.

+ The `Arr` class

Module `Data.Array` in stdio defined:

```haskell
class Arr (marr :: * -> * -> *) (arr :: * -> * ) a | arr -> marr, marr -> arr where
    newArr
    readArr
    writeArr  
    indexArr   
    ...
```

This is a type class trying to unify RTS's array interface, e.g. the `Data.Primitive.XXXArray` modules, it's multi-parameter class constraining both immutable and mutable array types. for example we have following instances:

```haskell
instance Arr MutableArray Array a where
instance Arr SmallMutableArray SmallArray a where
instance Prim a => Arr MutablePrimArray PrimArray a where
```

BTW, `PrimArray` is just a tagged version of `ByteArray` in `primitive` package:

```haskell
-- | The phantom type parameter a makes them an instance of `Arr`. 
newtype PrimArray a = PrimArray ByteArray
newtype MutablePrimArray s a = MutablePrimArray (MutableByteArray s)
```
Note this `Arr` class use functional-dependency to force an one-to-one immutable/mutable constrain, which is useful since lots of operations under `Arr` only mention either the immutable array type, or the mutable one.

+ The `Vec` class

Module `Data.Vector` in stdio defined:

```haskell
class (Arr (MArray v) (IArray v) a) => Vec v a where
    -- | Vector's mutable array type
    type MArray v :: * -> * -> *
    -- | Vector's immutable array type
    type IArray v :: * -> *
    -- | Get underline array and slice range(offset and length).
    toArr :: v a -> (IArray v a, Int, Int)
    -- | Create a vector by slicing an array(with offset and length).
    fromArr :: IArray v a -> Int -> Int -> v a
```

In `stdio`, we use `Vector/vector` to refer a slice of array, which is also the everyday data type we will be using. The `Vec` class here is how we constrain vector datatypes, we have following instance:

```haskell
data Vector a = Vector
    {-# UNPACK #-} !(SmallArray a) -- payload
    {-# UNPACK #-} !Int
    {-# UNPACK #-} !Int
    deriving (Typeable, Data)

instance Vec Vector a where ...

data PrimVector a = PrimVector
    {-# UNPACK #-} !(PrimArray a) -- payload
    {-# UNPACK #-} !Int         -- offset in elements of type a rather than in bytes
    {-# UNPACK #-} !Int         -- length in elements of type a rather than in bytes
  deriving (Typeable, Data)

instance Prim a => Vec PrimVector a where ...
```

`Vector a` is the boxed vector and `PrimVector a` is the unboxed one. Since our use case is immutable most of the time, we use `SmallArray` to reduce the overhead of updating card table.

To help working on vectors, we provide a pattern synonym:

```
pattern VecPat ba s l <- (toArr -> (ba,s,l))
```

+ `Text`

Module `Data.Text` in stdio defined:

```haskell
-- | In Data.Vector module we have this synonym
type Bytes = PrimVector Word8

newtype Text = Text { toUTF8Bytes :: V.Bytes }
```

`Text` is just a byte vector which must be UTF8-encoded, the only way to constructing such a value is to use:

```haskell
data UTF8DecodeResult
    = Success !Text
    | PartialBytes !Text !V.Bytes
    | InvalidBytes !V.Bytes

validateUTF8 :: V.Bytes -> UTF8DecodeResult
repairUTF8 :: V.Bytes -> (Text, V.Bytes)
```

That is, either by validation or doing bad codepoint repair(replacement). The encoder and decoder(in `Data.Text.Codec` module) is heavily optimized by hand which make this new UTF8-encoded `Text` type faster than the old UTF16 based text package.

I plan to add unicode case-mapping and normalization in future, and current plan is to use [utf8rewind](https://bitbucket.org/knight666/utf8rewind) package, which looks up to the task. But if anyone want to make a haskell version, i'll be happy to help out.

+ `Builder`

Module `Data.Builder` in stdio defined:

The `Builder` type is the standard way to constructing `PrimVector Word8`, which is defined as:

``` haskell
-- | 'AllocateStrategy' will decide how each 'BuildStep' proceed when previous buffer is not enough.
--
data AllocateStrategy m
    = DoubleBuffer       -- Double the buffer and continue building
    | InsertChunk {-# UNPACK #-} !Int   -- Insert a new chunk and continue building
    | OneShotAction (V.Bytes -> m ())   -- Freeze current chunk and perform action with it.
                                        -- Use the 'V.Bytes' argument outside the action is dangerous
                                        -- since we will reuse the buffer after action finished.

-- | Helper type to help ghc unpack
--
data Buffer s = Buffer {-# UNPACK #-} !(A.MutablePrimArray s Word8)  -- well, the buffer content
                       {-# UNPACK #-} !Int  -- writing offset

-- | @BuilderStep@ is a function that fill buffer under given conditions.
--
type BuildStep m = Buffer (PrimState m) -> m [V.Bytes]

-- | @Builder@ is a monoid to help compose @BuilderStep@. With next @BuilderStep@ continuation,
-- We can do interesting things like perform some action, or interleave the build process.
--
newtype Builder = Builder { runBuilder :: AllocateStrategy IO -> BuildStep IO -> BuildStep IO }
```

These types are just to make builder runs under all different demand, we have three ways to execute builder:

```haskell
buildBytes :: Builder -> V.Bytes
buildBytesList :: Builder -> [V.Bytes]
buildAndRun :: (V.Bytes -> IO ()) -> Builder -> IO ()
```

That is to directly turn builder into a byte vector, or turn it into a lazy list, or execute some IO action every time we fill a buffer up.

+ `Parser`

This part is under dev now

+ `FilePath`

This part is under dev now

+ `Handler`

This part is under dev now


Roadmap
-------

+ A unified vector type. (80%)
+ Unpinned bytestring based on vector. (80%) 
+ `Foregin` module for bytes. (0%)
+ IO system for `Bytes`, new `Handler` design. (10%)
+ IO system for `Bytes`, file and network part. (20%)
+ A compact `FilePath` type. (0%)
+ `Builder` for `Bytes`, both binary and textual. (50%)
+ `Parser` for `Bytes`, both binary and textual. (0%)
+ Basic UTF-8 text processing (0%)
+ Extend UTF-8 text processing, (normalization, unicode case-mapping, etc.) (0%)

Join in!
--------

This project need lots of effort than it seems, any contributions is welcome! Feel free to

+ Discuss design.
+ Report bugs.
+ Write implementations.

FAQ
---

+ Is this a custom `Prelude` thing?

No, this package only target a very limited focus: packed data types and IO interface, the vector, bytes, text, filepath, parser and builder parts are here to support. Anything out of this scope should not be included, for example, a better `Num` tower.

+ Why not use bytestring and text?

Bytestring is based on `ForeignPtr` which always allocate pinned memory, that can be slow and bring memory fragmentation. stdio use `ByteArray` based representation, which leave RTS's memory allocator to decide where to allocate. 

`Text` in stdio is UTF8-encoded.

+ Why not use vector?

One key design point is that stdio DO NOT use implicit stream fusion(both vector/bytes and text), but we provide optimized `pack/unpack` implementations which is good consumer/producer from foldr/build fusion perspective. which means if you need elimitate intermediate data structures you should do something like:

```haskell
pack . map YYY . filter XXX . unpack
```

`pack/unpack` has a little constant overhead but if the processing chain is long, it may be paid off. 

The reason for not introduce implicit fusion is that packed datatype is designed for sharing, which will destroy any form of fusion. On another hand, list in base is very bad at sharing but nice for processing. So in stdio the choice is simple: we only provide packed operations, `pack/unpack` when you want fusion to happen.

+ Why not foundation?

The array type in foundation is a sum type, which is unreasonable. And the abuse of type class/family is only a comlication IMO.

+ What about `mmap`?

Well, `mmap` certainly has its usage, but export `mmap` via pure data structures, e.g. `ByteString` is not a good idea IMO. Take a look at `System.IO.FD` module, the plan is to define a `MMap` data type and provide `FD/DiskFD` instance for it.

+ What about FFI with `Bytes` in stdio?

You just have to understand a little bit about GHC's RTS: an unsafe FFI call works like a fat primitive machine code op which stops GC. So basically you can pass `ByteArray` to any unsafe FFI calls with the help of `UnliftedFFITypes` extension. On the other hand, if you want to make a safe FFI call, use `isPrimArrayPinned` to decide if you want to allocate a pinned copy.

I plan to add a `Foregin.Bytes` module to help, which is not implemented yet.
