# fallible

An interface for fallible data types like Maybe and Either.

## Example

Dealing with Maybe and Either gets a bit annoying when monadic actions are
involved, often resulting in deep nests like this:

```Haskell
run :: String -> Token -> Bool -> IO ()
run targetName token verbose = do
  users <- getUsers token
  case users of
    Left err -> logDebug' err
    Right us -> do
      case userId <$> L.find isTarget us of
        Nothing  -> logDebug' emsg
        Just tid -> do
          channels <- getChannels token
          case channels of
            Left err  -> logDebug' err
            Right chs -> do
              let chs' = filter (elem tid . channelMembers) chs
              mapM_ (logDebug' . channelName) chs'
```

This package offers you several combinators to tidy up this eyesore. See
[example/Main.hs](example/Main.hs):

```Haskell
import           Data.Fallible
import qualified Data.List     as L

run :: String -> Token -> Bool -> IO ()
run targetName token verbose = evalContT $ do
  users    <- lift (getUsers token) !?= exit . logDebug'
  targetId <- userId <$> L.find isTarget users ??? exit (logDebug' emsg)
  channels <- lift (getChannels token) !?= exit . logDebug'
  lift $ mapM_ (logDebug' . channelName) $
    filter (elem targetId . channelMembers) channels
  where
    logDebug' = logDebug verbose
    emsg = "user not found: " ++ targetName
    isTarget user = userName user == targetName

logDebug :: Bool -> String -> IO ()
logDebug verbose msg = if verbose then putStrLn msg else pure ()
```

Notice the couple of operators, `(!?=)` and `(???)`:

```haskell
(!?=) :: Monad m => m (Either e a) -> (e -> m a) -> m a
(!??) :: Monad m => m (Maybe a) -> m a -> m a
(??=) :: Applicative f => Either e a -> (e -> f a) -> f a
(???) :: Applicative f => Maybe a -> f a -> f a
```

As you can guess from their type signatures, they run the right side when the
first arguments returns `Left` or `Nothing`. Then you can use whatever you like
to handle failures -- exception, MaybeT, ExceptT, ContT etc.

exec using ghci:

```
$ stack ghci
>>> :l example/Main.hs
*Main> run "Alice" "dummy" True
general
random
secret
*Main> run "Curry" "dummy" True
general
random
owners
*Main> run "Haskell" "dummy" True
user not found: Haskell
*Main> run "Haskell" "" True
invalid token
```

## Become a beta tester

### with Stack

write to stack.yaml:

```yaml
extra-deps:
  github: matsubara0507/fallible
  commit: xxx
```

### cabal

```
source-repository-package
  type: git
  location: https://github.com/matsubara0507/fallible
  tag: XXXXXXX
```
