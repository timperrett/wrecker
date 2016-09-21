This literate Haskell file is a simple walk through of how to use `wrecker` to benchmark a HTTP API.

Unlike most HTTP benchmarking applications, `wrecker` is intended to benchmark HTTP calls inline with other forms of processing. This allows for complex interactions necessary to benchmark certain API endpoints.

Before we get into the details we need to get some Haskell file setup out of the way.

First we turn on the extensions we would like to use.

```haskell
{-# LANGUAGE RecordPuns, DeriveAny, DeriveGeneric #-}
module WreckerSpec where
```

`RecordPuns` will let us destructure records conveniently. `DeriveAny` and `DeriveGeneric`
are used turned on so the compiler can generate the JSON conversion functions for us
automatically.

Now we import the packages necessary to make the client.

```haskell
import Wrecker (record, defaultMain)
```

`record` is the primary function from `wrecker`. It has the type `Recorder -> String -> IO a -> IO a`. `record` takes a `Recorder` and key in the form of a `String` and wraps some `IO` action. `record` runs the passed in `IO a` and um ... records information about such as the elapsed time and whether it succeeded or failed.


`defaultMain` is one of two entry points `wrecker` provides (the other is `run`). `defaultMain` performs command line argument parsing for us, and runs the benchmarks with the provided options. Additionally, `defaultMain` creates a `Recorder` that is used by all the benchmark scripts.

```haskell
import Data.Aeson
```
We need JSON  so of course we are using `aeson`.

```haskell
import qualified Network.Wreq as Wreq
```
`wrecker` does not provide any means for making HTTP calls. It records data, computes statistics, controls concurrency and provides a convenient UI. We leverage `wreq` to do the actual HTTP calls.

Here we wrap `wreq`'s `get` and `post` calls and make new functions which take a `Recorder` so we can benchmark the times.

```haskell
wreckerGet :: Recorder -> String -> String -> IO (Wreq.Response ByteString)
wreckerGet recorder key url = record recorder key (Wreq.get url)

wreckerPost :: Wreq.Postable a => String -> String -> a -> IO (Wreq.Response ByteString)
wreckerPost recorder key url x = record recorder key (Wreq.post url x)
```

Alright now it is time to make our client for out toy API.

For our client we will represent resource urls using the type `Ref`
```haskell
data Ref a = Ref { unRef :: Text }
  deriving (Show, Eq)
```

`Ref` is nothing more than a `Text` wrapper (the value there is the URL). `Ref` has polymorphic `a` so we can talk about different types of resources. It's use will become clearer later on.

```haskell
instance FromJSON (Ref a) where
  parseJSON = withText "FromJSON (Ref a)" Ref
```

A `FromJSON` instance which wraps a `Text` value, assuming the JSON is `Text`.

In addition to resources our API has ad-hoc RPC calls. RPC calls are also represented as a URL.

```haskell
data RPC a b = RPC String
  deriving (Show, Eq)

instance FromJSON (RPC a b) where
  parseJSON = withText "FromJSON (Ref a)" RPC
```

Next we utilize our

```haskell
fromEnvelope :: FromJSON a => IO (Wreq.Response ByteString) -> IO a
fromEnvelope x = fmap unEnvelope =<< asJSON x

get :: FromJSON a => Recorder -> String -> Ref a -> IO a
get recorder key (Ref url) = fromEnvelope =<< wreckerGet recorder key url

add :: (ToJSON a, FromJSON a) => Recorder -> String -> Ref [a] -> a -> IO ()
add recorder key (Ref url) x = void $ wreckerPost recorder key url $ toJSON x

rpc :: (ToJSON a, FromJSON b) => Recorder -> String -> RPC a b -> a -> IO b
rpc recorder key (RPC url) x = fromEnvelope
                           =<< wreckerPost recorder key url (toJSON x)
```

The API requires an initial call to the '/root' to obtain the URLs for subsequent calls

```haskell
rootRef :: Port -> Ref Root
rootRef port = Ref $ "http://localhost:" ++ show port ++ "/root"
```

Calling `GET` on `rootRef` returns the following JSON.

```json
{ "products" : "http://localhost:3000/products"
, "carts"    : "http://localhost:3000/carts"
, "users"    : "http://localhost:3000/users"
, "login"    : "http://localhost:3000/login"
, "checkout" : "http://localhost:3000/checkout"
}
```

We convert this to the following Haskell type:

```haskell
data Root = Root
  { products :: Ref [Ref Product]
  , carts    :: Ref [Ref Cart   ]
  , users    :: Ref [Ref User   ]
  , login    :: RPC Credentials (Ref User)
  , checkout :: RPC (Ref Cart)  ()
  } deriving (Eq, Show, Generic, FromJSON)
```

Since the JSON is so uniform, we can use 'aeson' generic instances.

```haskell
data Product = Product
  { summary :: Text
  } deriving (Eq, Show, Generic, FromJSON)
```

```haskell
data Cart = Cart
  { items :: Ref [Ref Product]
  } deriving (Eq, Show, Generic, FromJSON)
```

```haskell
data User = User
  { cart        :: Ref Cart
  , credentials :: Credentials
  } deriving (Eq, Show, Generic, FromJSON)
```

```haskell

```


```
testScript :: Port -> Recorder -> IO ()
testScript port recorder = do
  let get' = get recorder
      add' = add recorder
      rpc' = rpc recorder
```

Get the root relations and unpack the resource refs for further calls

```haskell

  Root {login, products} <- get' "root"     (rootRef port)
  userRef                <- rpc' "login"    login
                                             ( Credentials
                                               { userName = "a@example.com"
                                               , password = "password"
                                               }
                                             )
  User {cart }           <- get' "user"     userRef
  Cart {items}           <- get' "cart"     cart
```

```haskell
  -- We get all products and name the first one
  firstProduct : _       <- get' "products" products
  _                      <- add' "items"    items    firstProduct
  _                      <- rpc' "checkout" checkout cart

  return ()
```

Port is hard coded to 3000 for this example
```haskell
main :: IO ()
main = defaultMain [("test0", testScript 3000 port)]
```haskell