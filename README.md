envy
===================
Let's face it, dealing with environment variables in Haskell isn't that satisfying.

```haskell
import System.Environment
import Data.Text (pack)

data PGConfig = PGConfig {
  pgPort :: Int
  pgURL  :: Text
} deriving (Show, Eq)

getPGPort :: IO PGConfig
getPGPort = do
  portResult <- lookupEnv "PG_PORT"
  urlResult  <- lookupEnv "PG_URL"
  case (portResult, urlResult) of
    (Just port, Just url) ->
      case readMaybe port :: Maybe Int of
        Nothing -> error "PG_PORT isn't a number"
        Just portNum -> return $ PGConfig portNum (T.pack y)
    (Nothing, _) -> error "Couldn't find PG_PORT"    
    (_, Nothing) -> error "Couldn't find PG_URL"    
    -- Pretty gross right...
```

What if we could apply Aeson's FromJSON / ToJSON pattern to solve this problem?

Armed with the `GeneralizedNewTypeDeriving` extension we can derive instances of `Var` that will parse to and from an environment.

```haskell
{-# LANGUAGE RecordWildCards            #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE OverloadedStrings          #-}
------------------------------------------------------------------------------
module Main ( main ) where
------------------------------------------------------------------------------
import System.Environment
import Control.Monad
import System.Envy
import Test.Hspec
import Data.Text    (Text)
import Data.Either
------------------------------------------------------------------------------
-- | Posgtres port
newtype PGPORT = PGPORT Int
     deriving (Read, Show, Var)

------------------------------------------------------------------------------
-- | Postgres URL
newtype PGURL = PGURL String
     deriving (Read, Show, Var, IsString)

------------------------------------------------------------------------------
-- | Posgtres config
data PGConfig = PGConfig {
    pgPort :: PGPORT -- ^ Port 
  , pgURL  :: PGURL  -- ^ URL
  } deriving (Show, Read)

------------------------------------------------------------------------------
-- | Instances
instance FromEnv PGConfig where
  fromEnv env = do
    PGConfig <$> "PG_PORT" .: env 
             <*> "PG_URL"  .: env

------------------------------------------------------------------------------
-- | To Environment Instances
instance ToEnv PGConfig where
  toEnv PGConfig{..} =
       [ "PG_PORT" .= pgPort
       , "PG_URL"  .= pgURL
       ]

------------------------------------------------------------------------------
-- | Main
main :: IO ()
main = do 
  setEnvironment $ PGConfig 5432 "localhost" 
  print =<< do parseEnv :: IO (Either String PGConfig) 
  -- result: Right (PGConfig {pgPort = 5432, pgURL = "localhost"})
  -- unsetEnvironment $ PGConfig 5432 "localhost"  -- remove when done
```

Note: As of base 4.7 `setEnv` and `getEnv` throw an `IOException` if and `=` is present
in an environment. `envy` catches these synchronous exceptions and delivers them
purely to the end user.
