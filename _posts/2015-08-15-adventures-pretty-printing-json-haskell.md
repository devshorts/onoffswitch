---
layout: post
title: Adventures in pretty printing JSON in haskell
date: 2015-08-15 22:13:11.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- aeson
- haskell
- json
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558720432;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4244;}i:1;a:1:{s:2:"id";i:4327;}i:2;a:1:{s:2:"id";i:4348;}}}}

permalink: "/2015/08/15/adventures-pretty-printing-json-haskell/"
---
Today I gave atom haskell-ide a whirl and wanted to play with haskell a bit more. I've played with haskell in the past and always been put off by the tooling. To be fair, I still kind of am. I love the idea of the language but the tooling is just not there to make it an enjoyable exploratory experience. I spend half my time in the repl inspecting types, the other half on hoogle, and the 3rd half (yes I know) being frustrated that I can't just type in package names and explore API's in sublime or atom or wherever I am. Now that I'm on a mac, maybe I'll give leksah another try. I tried it a while ago it didn't work well.

Anyways, I digress. Playing with haskell and I thought I'd try poking with Aeson, the JSON library. Like Scala, you have to define your objects as json parseable/serializable (unlike java/c# which use runtime reflection). Thankfully if you enable some language extensions its just a matter of adding `Generic` to your derives list and making sure the data type is of the `ToJSON` and `FromJSON` typeclasses. Mostly I was just copying the examples from [here](https://www.fpcomplete.com/school/starting-with-haskell/libraries-and-frameworks/text-manipulation/json).

My sample class is

```haskell
  
{-# LANGUAGE DeriveGeneric #-}

module Types where

import Data.Aeson  
import GHC.Generics

data Person =  
 Person { firstName :: String  
 ,lastName :: String  
 } deriving(Show, Generic)

instance ToJSON Person  
instance FromJSON Person  

```

And I just wanted to make a simple hello world where I'd read in some data, make my object, and print pretty json to the screen.

On my first try:

```haskell
  
import Data.Aeson  
import Types

process :: IO String  
process = getLine

main = do  
 putStrLn "First name"  
 firstName \<- process

putStrLn "Last name"  
 lastName \<- process

let person = Person firstName lastName

print $ (encode person)

return ()  

```

When I run `cabal build;cabal run` I now get

```
  
First name  
anton  
Last name  
kropp  
"{\"lastName\":\"kropp\",\"firstName\":\"anton\"}"  

```

Certainly JSON, but I want it _pretty_. I found [aeson-pretty](https://hackage.haskell.org/package/aeson-pretty) and gave that a shot. Now I'm doing:

```haskell
  
import Data.Aeson  
import Data.Aeson.Encode.Pretty  
import Types

process :: IO String  
process = getLine

main = do  
 putStrLn "First name"  
 firstName \<- process

putStrLn "Last name"  
 lastName \<- process

let person = Person firstName lastName

print $ (encodePretty person)

return ()  

```

And I got:

```
  
First name  
anton  
Last name  
kropp  
"{\n \"lastName\": \"kropp\",\n \"firstName\": \"anton\"\n}"  

```

Hmm. I can see that it _should_ be pretty, but it isn't. How come? Lets check out the types:

```haskell
  
Prelude \> import Data.Aeson  
Prelude Data.Aeson \> :t encode  
encode :: ToJSON a =\> a -\> Data.ByteString.Lazy.Internal.ByteString  

```

Whats a lazy bytestring?

Well, from [fpcomplete](https://www.fpcomplete.com/school/to-infinity-and-beyond/pick-of-the-week/bytestring-bits-and-pieces)

> ByteString provides a more efficient alternative to Haskell's built-in String which can be used to store 8-bit character strings and also to handle binary data

And the lazy one is the, well, lazy version. Through some googling I find that the right way to print the bytestring is by using the "putStr" functions in the Data.ByteString package. But as a good functional programmer, I want to encapsulate that and basically make a useful function that given the json object I can get a plain ol happy string and decide how to print it later.

I need to somehow make a lazy bytestring into a regular string. This leads me to this:

```haskell
  
getJson :: ToJSON a =\> a -\> String  
getJson d = unpack $ decodeUtf8 $ BSL.toStrict (encodePretty d)  

```

So I first evaluate the bytestring into a strict version (instead of lazy), then decode it to utf8, then unpack the text class into strings (apparenlty text is more efficient but more API's use String).

```haskell
  
Prelude \> :t toStrict  
toStrict :: ByteString -\> Data.ByteString.Internal.ByteString

Prelude \> :t decodeUtf8  
decodeUtf8 :: Data.ByteString.Internal.ByteString -\> Text

Prelude \> :t Data.Text.unpack  
Data.Text.unpack :: Text -\> String  

```

And now, finally:

```haskell
  
import Data.Aeson  
import Data.Aeson.Encode.Pretty  
import qualified Data.ByteString.Lazy as BSL  
import Data.Text  
import Data.Text.Encoding  
import Types

process :: IO String  
process = getLine

getJson :: ToJSON a =\> a -\> String  
getJson d = unpack $ decodeUtf8 $ BSL.toStrict (encodePretty d)

main = do  
 putStrLn "First name"  
 firstName \<- process

putStrLn "Last name"  
 lastName \<- process

let person = Person firstName lastName

print $ getJson person

return ()  

```

Which gives me

```haskell
  
First name  
anton  
Last name  
kropp  
"{\n \"lastName\": \"kropp\",\n \"firstName\": \"anton\"\n}"  

```

AGHH! Still! Ok, more googling. Google google google.

Last piece of the puzzle is that print is really `putStrLn . show`

```haskell
  
Prelude \> let x = putStrLn . show  
Prelude \> x "foo\nbar"  
"foo\nbar"  

```

And if we just do

```haskell
  
Prelude \> putStrLn "foo\nbar"  
foo  
bar  

```

The missing ticket. Finally all put together:

```haskell
  
import Data.Aeson  
import Data.Aeson.Encode.Pretty  
import qualified Data.ByteString.Lazy as BSL  
import Data.Text  
import Data.Text.Encoding  
import Types

process :: IO String  
process = getLine

getJson :: ToJSON a =\> a -\> String  
getJson d = unpack $ decodeUtf8 $ BSL.toStrict (encodePretty d)

main = do  
 putStrLn "First name"  
 firstName \<- process

putStrLn "Last name"  
 lastName \<- process

let person = Person firstName lastName

putStrLn $ getJson person

return ()  

```

Which gives me:

```
  
$ cabal build; cabal run  
Building sample-0.1.0.0...  
Preprocessing executable 'sample' for sample-0.1.0.0...  
[3 of 3] Compiling Main ( src/Main.hs, dist/build/sample/sample-tmp/Main.o )  
Linking dist/build/sample/sample ...  
Preprocessing executable 'sample' for sample-0.1.0.0...  
Running sample...  
First name  
anton  
Last name  
kropp  
{  
 "lastName": "kropp",  
 "firstName": "anton"  
}  

```

Source available at my [github](https://github.com/devshorts/Playground/tree/master/haskell/aeson-tests)

