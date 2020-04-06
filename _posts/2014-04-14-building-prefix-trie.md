---
layout: post
title: Building a prefix trie
date: 2014-04-14 08:00:48.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- autocomplete
- haskell
- trie
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558781240;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4011;}i:1;a:1:{s:2:"id";i:3847;}i:2;a:1:{s:2:"id";i:3656;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2014/04/14/building-prefix-trie/"
---
[Prefix trie](http://en.wikipedia.org/wiki/Trie)'s are cool data structures that let you compress a dictionary of words based on their shared prefix. If you think about it, this makes a lot of sense. Why store `abs`, `abbr`, and `abysmal` when you only need to store `a,b,b,r,s,y,s,m,a,l`. Only storing what you have to (based on prefix) in this example gives you a 70% compression ratio! Not too bad, and it would only get better the more words you added.

The classical way of dealing with prefix tries is to store the suffixes using a map, but for fun I tried something different and used a list.

## Data

The main data structure I had was like this

[haskell]  
type Key a = [a]

data Trie key = Node (Maybe key) [Trie key] Bool deriving (Show, Eq, Read)  
[/haskell]

So a trie is really just a node that has a list of other tries (which would be its suffixes). You can see an example of the structure with the word "foo":

[haskell]  
[Node (Just 'f')  
 [Node (Just 'o')  
 [Node (Just 'o') [] True]  
 False]  
 False]  
[/haskell]

It's printable to the screen, equatable, and can be serialized from text. It also contains a boolean field representing if it is the end of a word or not. You can see the boolean at the last "o" of the word, indicating the word terminates. All the other characters are non-terminating fields.

## Searching

Let's add a helper method to find a key in a list of tries. Remember that each root can contain a list of other trie's, so it'll be nice to be able to say "_hey, we're at root X which has a list of possible suffix starts Y. Let's find the next root from the list of Y and return it_".

[haskell]  
{-  
 Finds a key in the trie list that matches the target key  
-}  
findKey :: (Eq t) =\> t -\> [Trie t] -\> Maybe (Trie t)  
findKey key tries = L.find (\(Node next \_ \_) -\> next == Just key) tries  
[/haskell]

It takes an equatable value, and a list of Trie's of those values, and finds the key returned as an option type.

We could even now write code to find a Trie

[haskell]  
{-  
 Takes a key list and finds the trie that fullfils that prefix  
-}  
findTrie :: (Eq t) =\> Key t -\> [Trie t] -\> Maybe (Trie t)  
findTrie [] \_ = Nothing  
findTrie (x:[]) tries = findKey x tries  
findTrie (x:xs) tries = findKey x tries \>\>= nextTrie  
 where nextTrie (Node \_ next \_) = findTrie xs next  
[/haskell]

Since `findKey` is an option type, we can leverage the option monadic bind operator `>>=`. This operator will pass a result to the next function only if the result is a `Just` type. This is a way to safely process 'nullable' items.

## Insertion

Now the insertion code, which honestly was the most complicated.

[haskell]  
insert :: (Eq t) =\> Key t -\> [Trie t] -\> [Trie t]  
insert [] t = t  
insert (x:xs) tries =  
 case findKey x tries of  
 Nothing -\> [(Node (Just x) (insert xs [])) isEndWord]++tries  
 Just value -\>  
 let (Node key next word) = value  
 in [Node key (insert xs next) (toggleWordEnd word)]++(except value)  
 where  
 except value = (L.filter ((/=) value) tries)  
 isEndWord = if xs == [] then True else False  
 toggleWordEnd old = if xs == [] then True else old  
[/haskell]

The trick to pure functional insertion is that you need to _rebuild_ the entire data structure WHILE you search for where you want to actually add things.

In the first case where you try to find a key but didn't find it, you want to add the key as a neighbor to the other trees. So if you have a root word of "_abd_" but you add "_def_", the root character of "_d_" doesn't exist yet and needs to be added, hence creating a new list and appending it to the original tries list. Then you recursively go through the new element and add the remaining list. This is why the creation of the new element has a recursive call back to the insert function with the remaining list `xs`. Also, since you created the new element as part of a recursive call, you know this element is already being properly appended to its root.

But, if the key already exists then that means you found a suffix that matches. At this point you need to bubble through the inner suffix tree to add the remaining items you want (basically following the suffix trie down to a point where you haven't already found a suffix root). Once thing to remember is that you also need to recreate the key you found, excluding the original element (hence the `except value` since we want to exclude the old version of the key before).

On top of that, if you've already processed a root, but your word now ends at a root, you need to toggle if that suffix is a word completion. For example, if I've already added "_abcdef_" then "_f_" is the end of the word. But now if I add "_abc_" as a word, then both "_c_" AND "_f_" are end words (so "_c_"'s end word boolean needs to be toggled to true).

I found this concept to be a little complicated because its both finding, updating, and rebuilding the data structure all at the same time, but with practice it does get more natural to trace through.

## Getting the words

Once you can build a trie though, we can leverage haskells list comprehensions to fold the words back out

[haskell]  
{-  
 Gives you all the available words in the trie list  
-}  
allWords :: [Trie b] -\> [[b]]  
allWords tries =  
 let raw = rawWords tries  
 in map (flatMap id) raw  
 where  
 flatMap f = Fold.concatMap (Fold.toList . f)  
 rawWords tries = [key:next  
 | (Node key suffixes isWord) \<- tries  
 , next \<-  
 if isWord then  
 []:(rawWords suffixes)  
 else  
 rawWords suffixes]  
[/haskell]

Since we have a list of roots, we can go through each suffix and append the root character to it. If we encounter a word, we can pop in an empty list to create a delimiter. The list comprehension will do every combination of every root with its suffixes, so we will get each word.

But, this makes a list of characters and so I wanted to flatmap the list to create strings.

## Guessing a suffix

We can even do auto complete stuff now! Let's look at how to guess what the available suffixes would be:

[haskell]  
{-  
 This function takes a key list and returns the  
 full words that match the key list. But, since we already  
 know about the source key that matches (the input) we can  
 prepend that information to any suffix information that is left.  
 If the node found that matches the original query is a word in itself  
 we can prepend that too since its a special case  
-}  
guess :: (Eq a) =\> Key a -\> [Trie a] -\> Maybe [Key a]  
guess word trie =  
 findTrie word trie \>\>= \(Node \_ next isWord) -\>  
 return $ (source isWord) ++ (prependOriginal word $ allWords next)  
 where  
 source isWord = if isWord then [word] else []  
 prependOriginal word = map (\elem -\> word ++ elem)  
[/haskell]

The idea is we have a string, and our original roots. We then look for the suffixes and when we find the target trie that ends at the word we have, we can find all the words after that and prepend our original word.

## Examples

Lets look at some basic examples of how to use the Trie:

[code]  
\*PrefixTree\> tries = build ["word", "wooordz", "happy", "hoppy"]

\*PrefixTree\> guess "h" tries  
Just ["hoppy","happy"]

\*PrefixTree\> allWords tries  
["hoppy","happy","wooordz","word"]

\*PrefixTree\> tries  
[Node (Just 'h') [Node (Just 'o') [Node (Just 'p') [Node (Just 'p') [Node (Just 'y') [] True] False] False] False,Node (Just 'a') [Node (Just 'p') [Node (Just 'p') [Node (Just 'y') [] True] False] False] False] False,Node (Just 'w') [Node (Just 'o') [Node (Just 'o') [Node (Just 'o') [Node (Just 'r') [Node (Just 'd') [Node (Just 'z') [] True] False] False] False] False,Node (Just 'r') [Node (Just 'd') [] True] False] False] False]

\*PrefixTree\> guess "ha" tries  
Just ["happy"]

\*PrefixTree\> guess "hi" tries  
Nothing  
[/code]

## Conclusion

It would be a lot faster to have leveraged a hash map instead of a list for the suffixes, since each time I need to find the suffix root I have to do an O(n) traversal, instead of an O(1), but since this isn't a production trie I'm not worried about it. It was still a fun exercise!

For a full example check my [github](https://github.com/devshorts/Playground/tree/master/haskell_trie).

