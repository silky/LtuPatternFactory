# Table of contents
- [Lambda the ultimate pattern factory](#lambda-the-ultimate-pattern-factory)
- [The Patternopedia](#the-patternopedia)
    - [Strategy -> Functor](#strategy---functor) 
    - [Singleton -> Pointed -> Applicative](#singleton---pointed---applicative) 
    - [Pipeline -> Monad](#pipeline---monad)
    - [NullObject -> Maybe Monad](#nullobject---maybe-monad)
    - [Composite -> SemiGroup -> Monoid](#composite---semigroup---monoid)
    - [Visitor -> Foldable](#visitor---foldable)
    - [Iterator -> Traversable](#iterator---traversable)
    - [Type classes Category, Arrow & Co.](#type-classes-category-arrow--co)
- [Beyond type class patterns](#beyond-type-class-patterns)
    - [Dependency Injection -> Parameter Binding](#dependency-injection---parameter-binding)
    - [Adapter -> Function Composition](#adapter---function-composition)
    - [Template Method -> type class default functions](#template-method---type-class-default-functions)
- [Some related links](#some-interesting-links)

# Lambda the Ultimate Pattern Factory

My first programming languages were Lisp, Scheme, and ML. When I later started to work in OO languages like C++ and Java I noticed that idioms that are standard vocabulary in functional programming (fp) were not so easy to achieve and required sophisticated structures. Books like [Design Patterns: Elements of Reusable Object-Oriented Software](https://en.wikipedia.org/wiki/Design_Patterns) were a great starting point to reason about those structures. One of my earliest findings was that several of the GoF-Patterns had a stark resemblance of structures that are built into in functional languages: for instance the strategy pattern corresponds to higher order functions in fp (more details see [below](#strategy)).

Recently, while re-reading through the [Typeclassopedia](https://wiki.haskell.org/Typeclassopedia) I thought it would be a good exercise to map the structure of software [design-patterns](https://en.wikipedia.org/wiki/Software_design_pattern#Classification_and_list) to the concepts found in the Haskell type class library and in functional programming in general.

By searching the web I found some blog entries studying specific patterns, but I did not come across any comprehensive study. As it seemed that nobody did this kind of work yet I found it worthy to spend some time on it and write down all my findings on the subject.

I think this kind of exposition could be helpful if you are either:
* a programmer with an OO background who wants to get a better grip on how to implement complexer designs in functional programming
* a functional programmer who wants to get a deeper intuition for type classes.

>This project is still work in progress, so please feel free to contact me with any corrections, adjustments, comments, suggestions and additional ideas you might have.
> Please use the [Issue Tracker](https://github.com/thma/LtuPatternFactory/issues) to enter your requests.
>
>Directions I'd like to cover in more depths are for instance:
> - complete coverage of the GOF set of patterns
> - coverage of category theory based patterns (any ideas are welcome!)
> - coverage of patterns with a clear FP background, eg. MapReduce, Blockchain, Function-as-a-service 

# The Patternopedia
The [Typeclassopedia](https://wiki.haskell.org/wikiupload/8/85/TMR-Issue13.pdf) is a now classic paper that introduces the Haskell type classes by clarifying their algebraic and category-theoretic background. In particular it explains the relationships among those type classes.

In this section I'm taking a tour through the Typeclassopedia from a design pattern perspective.
For each of the Typeclassopedia type classes (at least up to Traversable) I try to explain how it corresponds to structures applied in design patterns.


## Strategy -> Functor
> "The strategy pattern [...] is a behavioral software design pattern that enables selecting an algorithm at runtime. Instead of implementing a single algorithm directly, code receives run-time instructions as to which in a family of algorithms to use"

![strategy pattern](https://upload.wikimedia.org/wikipedia/commons/4/45/W3sDesign_Strategy_Design_Pattern_UML.jpg)

> "In the above UML class diagram, the `Context` class doesn't implement an algorithm directly. Instead, `Context` refers to the  `Strategy` interface for performing an algorithm (`strategy.algorithm()`), which makes `Context` independent of how an algorithm is implemented. The `Strategy1` and `Strategy2` classes implement the `Strategy` interface, that is, implement (encapsulate) an algorithm." 
>(quoted from https://en.wikipedia.org/wiki/Strategy_pattern)

* in C a strategy would be modelled as a function pointer that can be used to dispatch calls to different functions.
* In an OO language like Java a strategy would be modelled as a single strategy-method interface that would be implemented by different strategy classes that provide implementations of the strategy method.
* in functional programming a strategy is just a higher order function, that is a parameter of a function that has a function type.

```haskell
-- first we define two simple strategies that map numbers to numbers:
strategyId :: Num a => a -> a
strategyId n = n

strategyDouble :: Num a => a -> a
strategyDouble n = 2*n

-- now we define a context that applies a function of type Num a => a -> a to a list of a's:
context :: Num a => (a -> a) -> [a] -> [a]
context f l = map f l
-- according to the rules of currying this can be abbreviated to:
context = map
``` 
The `context` function uses higher order `map` function  (`map :: (a -> b) -> [a] -> [b]`) to apply the strategies to lists of numbers:
```haskell
ghci> context strategyId [1..10]
[1,2,3,4,5,6,7,8,9,10]
ghci> context strategyDouble [1..10]
[2,4,6,8,10,12,14,16,18,20]
```
Instead of map we could use just any other function that accepts a function of type `Num a => a -> a` and applies it in a given context.
In Haskell the application of a function in a computational context is generalized with the type class `Functor`:
```haskell
class  Functor f  where
    fmap :: (a -> b) -> f a -> f b
```
Actually `map` is the fmap implementation for the List Functor instance:
```haskell
instance Functor [] where
    fmap = map
```
Although it would be fair to say that the type class `Functor` captures the essential idea of the strategy pattern - namely the injecting into and the execution in a computational context of a function - the usage of higher order functions (or strategies) is of course not limited to `Functors` - we could use just any higher order function fitting our purpose. Other type classes like `Foldable` or `Traversable` can serve as helpful abstractions when dealing with typical use cases of applying variable strategies within a computational context. 

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/Strategy.hs)

## Singleton -> Pointed -> Applicative
> "The singleton pattern is a software design pattern that restricts the instantiation of a class to one object. This is useful when exactly one object is needed to coordinate actions across the system." 
> (quoted from https://en.wikipedia.org/wiki/Singleton_pattern)

The singleton pattern ensures that multiple requests to a given object always return one and the same singleton instance.
In functional programming this semantics can be achieved by ```let```. 

```haskell
let singleton = someExpensiveComputation
in  mainComputation

--or in lambda notation:
(\singleton -> mainComputation) someExpensiveComputation
```

Via the `let`-Binding we can thread the singleton through arbitrary code in the `in` block. All occurences of `singleton` in the `mainComputation`will point to the same instance.

Type classes provide several tools to make this kind of threading more convenient or even to avoid explicit threading of instances.

### Using Pointed to create singletons

> "Given a Functor, the Pointed class represents the additional ability to put a value into a “default context.” Often, this corresponds to creating a container with exactly one element, but it is more general than that." 
> (quoted from the Typeclassopedia)

```haskell
class Functor f => Pointed f where
    pure :: a -> f a
```



### Using Applicative Functor for threading of singletons

The following code defines a simple expression evaluator:

```haskell
data Exp e = Var String
           | Val e
           | Add (Exp e) (Exp e)
           | Mul (Exp e) (Exp e)

-- the environment is a list of tupels mapping variable names to values of type e
type Env e = [(String, e)] 

-- a simple evaluator reducing expression to numbers
eval :: Num e => Exp e -> Env e -> e
eval (Var x)   env = fetch x env
eval (Val i)   env = i
eval (Add p q) env = eval p env + eval q env  
eval (Mul p q) env = eval p env * eval q env 
```
`eval` is a classic evaluator function that recursively evaluates sub-expression before applying `+` or `*`.
Note how the explicit `env`parameter is threaded through the recursive eval calls. This is needed to have the 
environment avalailable for variable lookup at any recursive call depth.

If we now bind `env` to a value as in the following snippet it is used as an imutable singleton within the recursive evaluation of `eval exp env`.

```haskell
main = do
  let exp = Mul (Add (Val 3) (Val 1)) 
                (Mul (Val 2) (Var "pi"))
      env = [("pi", pi)]
  print $ eval exp env
```
Experienced Haskellers will notice the ["eta-reduction smell"](https://wiki.haskell.org/Eta_conversion) in `eval (Var x) env = fetch x env` which hints at the possibilty to remove `env` as an explicit parameter. We can not do this right away as the other equations for `eval` do not allow eta-reduction. In order to do so we have to apply the combinators of the `Applicative Functor`:
```haskell
class Functor f => Applicative f where
    pure  :: a -> f a
    (<*>) :: f (a -> b) -> f a -> f b

instance Applicative ((->) a) where
    pure        = const
    (<*>) f g x = f x (g x)
```
This `Applicative` allows us to rewrite `eval` as follows:
```haskell
eval :: Num e => Exp e -> Env e -> e
eval (Var x)   = fetch x
eval (Val i)   = pure i
eval (Add p q) = pure (+) <*> eval p  <*> eval q  
eval (Mul p q) = pure (*) <*> eval p  <*> eval q 
```
Any explicit handling of the variable `env` is now removed.
(I took this example from the classic paper [Applicative programming with effects](http://www.soi.city.ac.uk/~ross/papers/Applicative.pdf) which details how `pure` and `<*>` correspond to the combinatory logic combinators `K` and `S`.)

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/Singleton.hs)

## Pipeline -> Monad
> In software engineering, a pipeline consists of a chain of processing elements (processes, threads, coroutines, functions, etc.), arranged so that the output of each element is the input of the next; the name is by analogy to a physical pipeline.
> (Quoted from: https://en.wikipedia.org/wiki/Pipeline_(software))

The concept of pipes and filters in Unix shell scripts is a typical example of the pipeline architecture pattern.
```bash
$ echo "hello world" | wc -w | xargs printf "%d*3\n" | bc -l
6
``` 
This works exactly as stated in the wikipedia definition of the pattern: the output of `echo "hello world"` is used as input for the next command `wc -w`. The ouptput of this command is then piped as input into `xargs printf "%d*3\n"` and so on.
On the first glance this might look like ordinary function composition. We could for instance come up with the following approximation in Haskell:
```haskell
((3 *) . length . words) "hello world"
6
```
But with this design we missed an important feature of the chain of shell commands: The commands do not work on elementary types like Strings or numbers but on input and output streams that are used to propagate the actual elementary data around. So we can't just send a String into the `wc` command as in `"hello world" | wc -w`. Instead we have to use `echo` to place the string into a stream that we can then use as input to the `wc` command:
```bash
$ echo "hello world" | wc -w
```
So we might say that `echo` *injects* the String `"hello world"` into the stream context.
We can capture this behaviour in a functional program like this:
```haskell
-- The Stream type is a wrapper around an arbitrary payload type 'a'
newtype Stream a = Stream a deriving (Show)

-- echo injects an item of type 'a' into the Stream context
echo :: a -> Stream a
echo = Stream

-- the 'andThen' operator used for chaining commands
infixl 7 |>
(|>) :: Stream a -> (a -> Stream b) -> Stream b
Stream x |> f = f x


-- echo and |> are used to create the actual pipeline
pipeline :: String -> Stream Int
pipeline str = 
  echo str |> echo . length . words |> echo . (3 *)
-- now executing the program in ghci repl:
ghci> pipeline "hello world"
Stream 6  
```

The `echo` function injects any input into the `Stream` context:
```haskell
ghci> echo "hello world"
Stream "hello world"
```
The `|>` (pronounced as "andThen") does the function chaining:
```haskell
ghci> echo "hello world" |> echo . words
Stream ["hello","world"]
```
The result of `|>` is of type `Stream b` that's why we cannot just write `echo "hello world" |> words`. We have to use echo  to create a `Stream` output that can be digested by a subsequent `|>`.

The interplay of a Context type `Stream a` and the functions `echo` and `|>` is a well known pattern from functional languages: it's the legendary *Monad*. As the [Wikipedia article on the pipeline pattern](https://en.wikipedia.org/wiki/Pipeline_(software)) states:

> Pipes and filters can be viewed as a form of functional programming, using byte streams as data objects; more specifically, they can be seen as a particular form of monad for I/O. 

There is an interesting paper available elaborating on the monadic nature of Unix pipes: http://okmij.org/ftp/Computation/monadic-shell.html.

Here is the definition of the Monad type class in Haskell:
```Haskell
class Applicative m => Monad m where
    -- | Sequentially compose two actions, passing any value produced
    -- by the first as an argument to the second.
    (>>=)  :: m a -> (a -> m b) -> m b

    -- | Inject a value into the monadic type.
    return :: a -> m a
    return = pure
```
By looking at the types of `>>=` and `return` it's easy to see the direct correspondence to `|>` and `echo` in the pipeline example above:
```haskell
    (|>)   :: Stream a -> (a -> Stream b) -> Stream b
    echo   :: a -> Stream a
```

Mhh, this is nice, but still looks a lot like ordinary composition of functions, just with the addition of a wrapper. 
In this simplified example that's true, because we have designed the `|>` operator to simply unwrap a value from the Stream and bind it to the formal parameter of the subsequent function:
```haskell
Stream x |> f = f x
```
But we are free to implement the `andThen` operator in any way that we seem fit as long we maintain the type signature and the [monad laws](https://en.wikipedia.org/wiki/Monad_%28functional_programming%29#Monad_laws).
So we could for instance change the semantic of `>>=` to keep a log along the execution pipeline.
In the following snippet I have extended `>>=` to increment a counter so that at the and of the pipeline we are informed about the number of invocations of `>>=`.

```haskell
-- The DeriveFunctor Language Pragma provides automatic derivation of Functor instances
{-# LANGUAGE DeriveFunctor #-}
-- the Stream type is extened by an Int that keeps the counter state
newtype Stream a = Stream (a, Int) deriving (Show, Functor)

-- as any Monad must be an Applicative we also have to instantiate Applicative
instance Applicative Stream where
  pure = return
  Stream (f, _) <*> r = fmap f r

-- our definition of the Stream Monad
instance Monad Stream where
  -- returns a Stream wrapping a tuple of the actual payload and an initial counter state of 0
  return a = Stream (a, 0)
  -- we define (>>=) to reach an incremented counter to the subsequent action
  m >>= k = let Stream(a, c1) = m
                next          = k a
                Stream(b, c2) = next
            in Stream (b, c1 + 1 + c2)

-- instead of echo and |> we now use the "official" monadic versions return and >>=
pipeline :: String -> Stream Int
pipeline str =
  return str >>= return . length . words >>= return . (3 *)

-- when using this in GHCI we receive a Stream wrapping a tuple of the result of the 
-- actual pipeline plus the result of the counter:
ghci> pipeline "hello world"
Stream (6,2)
```
What's noteworthy here is that Monads allow to make the mechanism of chaining functions *explicit*. We can define what `andThen` should mean in our pipeline by choosing a different Monad implementation.
So in a sense Monads could be called [programmable semicolons](http://book.realworldhaskell.org/read/monads.html#id642960)

To make this statement a bit clearer we will have a closer look at the internal workings of the `Maybe` Monad in the next section.

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/Pipeline.hs)

## NullObject -> Maybe Monad

>[...] a null object is an object with no referenced value or with defined neutral ("null") behavior. The null object design pattern describes the uses of such objects and their behavior (or lack thereof).
> [Quoted from Wikipedia](https://en.wikipedia.org/wiki/Null_object_pattern)

In functional programming the null object pattern is typically formalized with option types:
> [...] an option type or maybe type is a polymorphic type that represents encapsulation of an optional value; e.g., it is used as the return type of functions which may or may not return a meaningful value when they are applied. It consists of a constructor which either is empty (named None or `Nothing`), or which encapsulates the original data type `A` (written `Just A` or Some A).
> [Quoted from Wikipedia](https://en.wikipedia.org/wiki/Option_type)

(See also: [Null Object as Identity](http://blog.ploeh.dk/2018/04/23/null-object-as-identity/))

In Haskell the most simple option type is `Maybe`. Let's directly dive into an example. We define a reverse index, mapping songs to album titles.
If we now lookup up a song title we may either be lucky and find the respective album or not so lucky when there is no album matching our song:

```haskell
import           Data.Map (Map, fromList) 
import qualified Data.Map as Map (lookup) -- avoid clash with Prelude.lookup

-- type aliases for Songs and Albums
type Song   = String
type Album  = String

-- the simplified reverse song index
songMap :: Map Song Album
songMap = fromList
    [("Baby Satellite","Microgravity")
    ,("An Ending", "Apollo: Atmospheres and Soundtracks")]

```
We can lookup this map by using the function `Map.lookup :: Ord k => k -> Map k a -> Maybe a`.

If no match is found it will return `Nothing` if a match is found it will return `Just match`:

```haskell
ghci> Map.lookup "Baby Satellite" songMap
Just "Microgravity"
ghci> Map.lookup "The Fairy Tale" songMap
Nothing
```
Actually the `Maybe` type is defined as:
```haskell
data  Maybe a  =  Nothing | Just a 
    deriving (Eq, Ord)
```
All code using the `Map.lookup` function will never be confronted with any kind of Exceptions, null pointers or other nasty things. Even in case of errors a lookup will always return a properly typed `Maybe` instance. By pattern matching for `Nothing` or `Just a` client code can react on failing matches or positive results:

```haskell
    case Map.lookup "Ancient Campfire" songMap of
        Nothing -> print "sorry, could not find your song"
        Just a  -> print a
```
Let's try to apply this to an extension of our simple song lookup.
Let's assume that our music database has much more information available. Apart from a reverse index from songs to albums, there might also be an index mapping album titles to artists.
And we might also have an index mapping artist names to their websites:
```haskell
type Song   = String
type Album  = String
type Artist = String
type URL    = String

songMap :: Map Song Album
songMap = fromList
    [("Baby Satellite","Microgravity")
    ,("An Ending", "Apollo: Atmospheres and Soundtracks")]

albumMap :: Map Album Artist
albumMap = fromList
    [("Microgravity","Biosphere")
    ,("Apollo: Atmospheres and Soundtracks", "Brian Eno")]    

artistMap :: Map Artist URL
artistMap = fromList
    [("Biosphere","http://www.biosphere.no//")
    ,("Brian Eno", "http://www.brian-eno.net")]    

loookup' :: Ord a => Map a b -> a -> Maybe b
loookup' = flip Map.lookup    

findAlbum :: Song -> Maybe Album
findAlbum = loookup' songMap 

findArtist :: Album -> Maybe Artist
findArtist = loookup' albumMap

findWebSite :: Artist -> Maybe URL
findWebSite = loookup' artistMap
```

With all this information at hand we want to write a function that has an input parameter of type `Song` and returns a `Maybe URL` by going from song to album to artist to website url:
```haskell
findUrlFromSong :: Song -> Maybe URL
findUrlFromSong song = 
    case findAlbum song of
        Nothing    -> Nothing
        Just album -> 
            case findArtist album of
                Nothing     -> Nothing
                Just artist ->
                    case findWebSite artist of
                        Nothing  -> Nothing
                        Just url -> Just url
```
This code makes use of the pattern matching logic described before. It's worth to note that there is some nice circuit breaking happening in case of a `Nothing`. In this case `Nothing` is directly returned as result of the function and the rest of the case-ladder is not executed.
What's not so nice is *"the dreaded ladder of code marching off the right of the screen"* [(quoted from Real World Haskell)](http://book.realworldhaskell.org/).

For each find function we have to repeat the same ceremony of pattern matching on the result and either return `Nothing` or proceed with the next nested level. 

The good news is that it is possible to avoid this ladder.
We can rewrite our search by applying the `andThen` operator `>>=` as `Maybe` is an instance of `Monad`:
```haskell
findUrlFromSong' :: Song -> Maybe URL
findUrlFromSong' song =
    findAlbum song   >>= \album ->
    findArtist album >>= \artist ->
    findWebSite artist  
```
or even shorter as we can eliminate the lambda expressions by applying [eta-conversion](https://wiki.haskell.org/Eta_conversion):
```haskell
findUrlFromSong'' :: Song -> Maybe URL
findUrlFromSong'' song =
    findAlbum song >>= findArtist >>= findWebSite 
```
Using it in GHCi:
```haskell
ghci> findUrlFromSong'' "All you need is love"
Nothing
ghci> findUrlFromSong'' "An Ending"
Just "http://www.brian-eno.net"
```

The expression `findAlbum song >>= findArtist >>= findWebSite` and the sequencing of actions in the [pipeline](#pipeline---monad) example `return str >>= return . length . words >>= return . (3 *)` have a similar structure.

But the behaviour of both chains is quite different: In the Maybe Monad `a >>= b` does not evaluate b if `a == Nothing` but stops the whole chain of actions by simply returning `Nothing`. 

The pattern matching and 'short-circuiting' is directly coded into the definition of `(>>=)` in the Monad implementation of `Maybe`:
```haskell
instance  Monad Maybe  where
    (Just x) >>= k      = k x
    Nothing  >>= _      = Nothing
```

This elegant feature of `(>>=)` in the `Maybe` Monad allows us to avoid ugly and repetetive coding.  

There are several predefined Monads available in the Haskell curated libraries and it's also possible to combine their effects by making use of `MonadTransformers`. But that's a different story...

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/NullObject.hs)

<!-- 
#### TBD: Reimplementing the Evaluator with Writer-Monad
-->

## Composite -> SemiGroup -> Monoid

>In software engineering, the composite pattern is a partitioning design pattern. The composite pattern describes a group of objects that is treated the same way as a single instance of the same type of object. The intent of a composite is to "compose" objects into tree structures to represent part-whole hierarchies. Implementing the composite pattern lets clients treat individual objects and compositions uniformly.
> (Quoted from [Wikipedia](https://en.wikipedia.org/wiki/Composite_pattern))

A typical example for the composite pattern is the hierarchical grouping of test cases to TestSuites in a testing framework. Take for instance the following class diagram from the [JUnit cooks tour](http://junit.sourceforge.net/doc/cookstour/cookstour.htm) which shows how JUnit applies the Composite pattern to group `TestCases` to `TestSuites` while both of them implement the `Test` interface:

![Composite Pattern used in Junit](http://junit.sourceforge.net/doc/cookstour/Image5.gif)


In Haskell we could model this kind of hierachy with an algebraic data type (ADT):

```haskell
-- the composite data structure: a Test can be either a single TestCase
-- or a TestSuite holding a list of Tests
data Test = TestCase TestCase
          | TestSuite [Test]

-- a test case produces a boolean when executed
type TestCase = () -> Bool
```
The function `run` as defined below can either execute a single TestCase or a composite TestSuite:
```haskell
-- execution of a Test. 
run :: Test -> Bool
run (TestCase t)  = t () -- evaluating the TestCase by applying t to ()
run (TestSuite l) = all (True ==) (map run l) -- running all tests in l and return True if all tests pass

-- a few most simple test cases    
t1 :: Test    
t1 = TestCase (\() -> True)
t2 :: Test 
t2 = TestCase (\() -> True)
t3 :: Test 
t3 = TestCase (\() -> False)
-- collecting all test cases in a TestSuite
ts = TestSuite [t1,t2,t3]
```
As run is of type `run :: Test -> Bool` we can use it to execute single `TestCases` or complete `TestSuites`. 
Let's try it in GHCI:
```
ghci> run t1
True
ghci> run ts
False
```

In order to aggregate TestComponents we follow the design of JUnit and define a function `addTest`. Adding two atomic Tests will result in a TestSuite holding a list with the two Tests. If a Test is added to a TestSuite, the test is added to the list of tests of the suite. Adding TestSuites will merge them.
```haskell
-- adding Tests
addTest :: Test -> Test -> Test
addTest t1@(TestCase _) t2@(TestCase _)   = TestSuite [t1,t2]
addTest t1@(TestCase _) (TestSuite list)  = TestSuite ([t1] ++ list)
addTest (TestSuite list) t2@(TestCase _)  = TestSuite (list ++ [t2])
addTest (TestSuite l1) (TestSuite l2)     = TestSuite (l1 ++ l2)
```

If we take a closer look at `addTest` we will see that it is a associative binary operation on the set of `Test`s. 

In mathemathics a set with an associative binary operation is a [Semigroup](https://en.wikipedia.org/wiki/Semigroup).

We can thus make our type `Test` an instance of the type class `Semigroup` with the following declaration:

```haskell
instance Semigroup Test where
    (<>)   = addTest
```
What's not visible from the JUnit class diagram is how typical object oriented implementations will have to deal with null-references. That is the implementations would have to make sure that the methods `run` and `addTest` will handle empty references correctly. 
With Haskells algebraic data types we would rather make this explicit with a dedicated `Empty` element.
Here are the changes we have to add to our code:
```haskell
-- the composite data structure: a Test can be Empty, a single TestCase
-- or a TestSuite holding a list of Tests
data Test = Empty
          | TestCase TestCase
          | TestSuite [Test]

-- execution of a Test. 
run :: Test -> Bool
run Empty         = True -- empty tests will pass
run (TestCase t)  = t () -- evaluating the TestCase by applying t to ()
--run (TestSuite l) = foldr ((&&) . run) True l
run (TestSuite l) = all (True ==) (map run l) -- running all tests in l and return True if all tests pass

-- addTesting Tests
addTest :: Test -> Test -> Test
addTest Empty t                           = t
addTest t Empty                           = t
addTest t1@(TestCase _) t2@(TestCase _)   = TestSuite [t1,t2]
addTest t1@(TestCase _) (TestSuite list)  = TestSuite ([t1] ++ list)
addTest (TestSuite list) t2@(TestCase _)  = TestSuite (list ++ [t2])
addTest (TestSuite l1) (TestSuite l2)     = TestSuite (l1 ++ l2)
```
From our additions it's obvious that `Empty` is the identity element of the `addTest` function. In Algebra a Semigroup with an identity element is called *Monoid*:

> In abstract algebra, [...] a monoid is an algebraic structure with a single associative binary operation and an identity element. 
> [Quoted from Wikipedia](https://en.wikipedia.org/wiki/Monoid)


With haskell we can declare `Test` as an instance of the `Monoid` type class by defining:
```haskell
instance Monoid Test where
    mempty = Empty
```
We can now use all functions provided by the `Monoid` type class to work with our `Test`:

```haskell
compositeDemo = do
    print $ run $ t1 <> t2
    print $ run $ t1 <> t2 <> t3
```
We can also use the function `mconcat :: Monoid a => [a] -> a` on a list of `Tests`: mconcat composes a list of Tests into a single Test. That's exactly the mechanism of forming a TestSuite from atomic TestCases.
```haskell
compositeDemo = do
    print $ run $ mconcat [t1,t2]
    print $ run $ mconcat [t1,t2,t3]
```
This particular feature of `mconcat :: Monoid a => [a] -> a` to condense a list of Monoids to a single Monoid can be used to drastically simplify the design of our test framework.

We need just one more hint from our mathematician friends:

> Functions are monoids if they return monoids
> [Quoted from blog.ploeh.dk](http://blog.ploeh.dk/2018/05/17/composite-as-a-monoid-a-business-rules-example/)

Currently our `TestCases` are defined as functions yielding boolean values:
```haskell
type TestCase = () -> Bool
```
If `Bool` was a `Monoid` we could use `mconcat` to form test suite aggregates. `Bool` in itself is not a Monoid; but together with a binary associative operation like `(&&)` or `(||)` it will form a Monoid. 

The intuitive semantics of a TestSuite is that a whole Suite is "green" only when all enclosed TestCases succeed. That is the conjunction of all TestCases must return `True`.

 So we are looking for the Monoid of boolean values under conjunction `(&&)`. In Haskell this Monoid is called `All`):

```haskell
-- | Boolean monoid under conjunction ('&&').
-- >>> getAll (All True <> mempty <> All False)
-- False
-- >>> getAll (mconcat (map (\x -> All (even x)) [2,4,6,7,8]))
-- False
newtype All = All { getAll :: Bool }

instance Semigroup All where
        (<>) = coerce (&&)

instance Monoid All where
        mempty = All True
```

Making use of `All` our improved definition of TestCases is as follows:
```haskell
type SmartTestCase = () -> All
```
Now our test cases do not directly return a boolean value but an `All` wrapper, which allows automatic conjunction of test results to a single value. 
Here are our redefined TestCases:
```haskell
tc1 :: SmartTestCase
tc1 () = All True
tc2 :: SmartTestCase
tc2 () = All True
tc3 :: SmartTestCase
tc3 () = All False
```
We now implement a new evaluation function `run'` which evaluates a `SmartTestCase` (which may be either an atomic TestCase or a TestSuite assembled by `mconcat`) to a single boolean result.

```haskell
run' :: SmartTestCase -> Bool
run' tc = getAll $ tc ()  
```

This version of `run` is much simpler than the original and we can completely avoid the rather laborious `addTest` function. We also don't need any composite type `Test`.
By just sticking to the Haskell built-in type classes we achieve cleanly designed functionality with just a few lines of code.

```haskell
compositeDemo = do
    -- execute a single test case
    print $ run' tc1

    --- execute a complex test suite
    print $ run' $ mconcat [tc1,tc2]
    print $ run' $ mconcat [tc1,tc2,tc3]
```
For more details on Composite as a Monoid please refer to the following blog:
http://blog.ploeh.dk/2018/03/12/composite-as-a-monoid/

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/Composite.hs)

## Visitor -> Foldable

> [...] the visitor design pattern is a way of separating an algorithm from an object structure on which it operates. A practical result of this separation is the ability to add new operations to existent object structures without modifying the structures. It is one way to follow the open/closed principle.
> (Quoted from [Wikipedia](https://en.wikipedia.org/wiki/Visitor_pattern))


In functional languages - and Haskell in particular - we have a whole armada of tools serving this purpose:
* higher order functions like map, fold, filter and all their variants allow to "visit" lists
* The Haskell type classes `Functor`, `Foldable`, `Traversable`, etc. provide a generic framework to allow visiting any algebraic datatype by just deriving one of these type classes.
### Using Foldable
```haskell
-- we are re-using the Exp data type from the Singleton example 
-- and transform it into a Foldable type:
instance Foldable Exp where
    foldMap f (Val x)   = f x
    foldMap f (Add x y) = foldMap f x `mappend` foldMap f y
    foldMap f (Mul x y) = foldMap f x `mappend` foldMap f y

filterF :: Foldable f => (a -> Bool) -> f a -> [a]
filterF p = foldMap (\a -> if p a then [a] else [])     

visitorDemo = do
    let exp = Mul (Add (Val 3) (Val 2)) 
                  (Mul (Val 4) (Val 6))
    putStr "size of exp: "
    print $ length exp
    putStrLn "filter even numbers from tree"
    print $ filterF even exp
```
By virtue of the instance declaration Exp becomes a Foldable instance an can be used with arbitrary functions defined on Foldable like `length` in the example.

`foldMap` can for example be used to write a filtering function `filterF`that collects all elements matching a predicate into a list. 

#### Alternative approaches
http://blog.ploeh.dk/2018/06/25/visitor-as-a-sum-type/

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/Visitor.hs)

## Iterator -> Traversable

> [...] the iterator pattern is a design pattern in which an iterator is used to traverse a container and access the container's elements. The iterator pattern decouples algorithms from containers; in some cases, algorithms are necessarily container-specific and thus cannot be decoupled. 
> [Quoted from Wikipedia] (https://en.wikipedia.org/wiki/Iterator_pattern)

### Iterating over a Tree

The most generic type class enabling iteration over algebraic data types is `Traversable` as it allows combinations of `map` and `fold` operations.
We are re-using the `Exp` type from earlier examples to show what's needed for enabling iteration in functional languages.

```haskell
instance Functor Exp where
    fmap f (Var x)       = Var x
    fmap f (Val a)       = Val $ f a
    fmap f (Add x y)     = Add (fmap f x) (fmap f y)
    fmap f (Mul x y)     = Mul (fmap f x) (fmap f y)

instance Traversable Exp where
    traverse g (Var x)   = pure $ Var x
    traverse g (Val x)   = Val <$> g x
    traverse g (Add x y) = Add <$> traverse g x <*> traverse g y
    traverse g (Mul x y) = Mul <$> traverse g x <*> traverse g y
```
With this declaration we can traverse an `Exp` tree:
```haskell
iteratorDemo = do
    putStrLn "Iterator -> Traversable"
    let exp = Mul (Add (Val 3) (Val 1)) 
                (Mul (Val 2) (Var "pi"))
        env = [("pi", pi)]
    print $ traverse (\x c -> if even x then [x] else [2*x]) exp 0
```
In this example we are touching all (nested) `Val` elements and multiply all odd values by 2.

### Combining traversal operations

Compared with `Foldable` or `Functor` the declaration of a `Traversable` instance looks a bit intimidating. In particular the type declaration for `traverse`:
```haskell
traverse :: (Traversable t, Applicative f) => (a -> f b) -> t a -> f (t b)
```
looks like quite a bit of over-engineering for simple traversals as in the above example.

In oder to explain real power of the `Traversable` type class we will look at a more sophisticated example in this section.

The Unix utility `wc` is a good example for a traversal operation that performs several different tasks while traversing its input:

```bash
echo "counting lines, words and characters in one traversal" | wc
      1       8      54
```
The output simply means that our input has 1 line, 8 words and a total of 54 characters.
Obviously an efficients implementation of `wc` will accumulate the three counters for lines, words and characters in a single pass of the input will not run three iterations for each counter separately.

Here is a Java implementation:

```java
private static int[] wordCount(String str) {
    int nl=0, nw=0, nc=0;         // number of lines, number of words, number of characters
    boolean readingWord = false;  // state information for "parsing" words
    for (Character c : asList(str)) {
        nc++;                     // count just any character
        if (c == '\n') {
            nl++;                 // count only newlines
        }
        if (c == ' ' || c == '\n' || c == '\t') {
            readingWord = false;  // when detecting white space, signal end of word
        } else if (readingWord == false) {
            readingWord = true;   // when switching from white space to characters, signal new word
            nw++;                 // increase the word counter only once while in a word
        }
    }
    return new int[]{nl,nw,nc};
}

private static List<Character> asList(String str) {
    return str.chars().mapToObj(c -> (char) c).collect(Collectors.toList());
}
```
Please note that the `for (Character c : asList(str)) {...}` notation is just syntactic sugar for 

```java
for (Iterator<Character> iter = asList(str).iterator(); iter.hasNext();) {
    Character c = iter.next();
    ...
}     
```
For efficiency reasons this solution may be okay, but from a design perspective the solution lacks clarity as the required logic for accumulating the three counters is heavily entangled within one code block. Just imagine how the complexity of the for-loop will increase once we have to add new features like counting bytes, counting white space or counting maximum line width.

So we would like to be able to isolate the different counting algorithms (*separation of concerns*) and be able to combine them in a way that provides efficient one-time traversal.

We start with the simple task of character counting:
```haskell
type Count = Const (Sum Integer)

count :: a -> Count b
count _ = Const 1

cciBody :: Char -> Count a
cciBody = count

cci :: String -> Count [a]
cci = traverse cciBody

-- and then in ghci:
> cci "hello world"
Const (Sum {getSum = 11})
```
For each character we just emit a `Const 1` which are elements of type `Const (Sum Integer)`.
As `(Sum Integer)` is the monoid of Integers under addition, this design allows automatic summation over all collected `Const` values.

The next step of counting newlines looks similar:
```haskell
-- return (Sum 1) if true, else (Sum 0)
test :: Bool -> Sum Integer
test b = Sum $ if b then 1 else 0

-- use the test function to emit (Sum 1) only when a newline char is detected
lciBody :: Char -> Count a
lciBody c = Const $ test (c == '\n')

-- define the linecount using traverse
lci :: String -> Count [a]
lci = traverse lciBody

-- and the in ghci:
> lci "hello \n world"
Const (Sum {getSum = 1})
```
Now let's try to combine character counting and line counting.
In order to match the type declaration for `traverse`:
```haskell
traverse :: (Traversable t, Applicative f) => (a -> f b) -> t a -> f (t b)
```
we had to define `cciBody` and `lciBody` so that their return types are `Applicative Functors`.
The good news is that the product of two `Applicatives` is again an `Applicative` (the same holds true for Composition of `Applicatives`).
With this knowledge we can now use `traverse` to use the product of `cciBody` and `lciBody`:

```haskell
import Data.Functor.Product             -- Product of Functors

-- define infix operator for building a Functor Product
(<#>) :: (Functor m, Functor n) => (a -> m b) -> (a -> n b) -> (a -> Product m n b)
(f <#> g) y = Pair (f y) (g y) 

-- use a single traverse to apply the Product of cciBody and lciBody
clci :: String -> Product Count Count [a]
clci = traverse (cciBody <#> lciBody)

-- and then in ghci:
> clci "hello \n world"
Pair (Const (Sum {getSum = 13})) (Const (Sum {getSum = 1}))
```
So we have achieved our aim of separating line counting and character counting in separate functions while still being able to apply them in only one traversal.

The only piece missing is the word counting. This is a bit tricky as it involves dealing with a state monad and wrapping it as an Applicative Functor:

```haskell
import Data.Functor.Compose             -- Composition of Functors
import Data.Functor.Const               -- Const Functor
import Data.Functor.Identity            -- Identity Functor (needed for coercion)
import Data.Monoid (Sum (..), getSum)   -- Sum Monoid for Integers
import Control.Monad.State.Lazy         -- State Monad
import Control.Applicative              -- WrappedMonad (wrapping a Monad as Applicative Functor)
import Data.Coerce (coerce)             -- Coercion (forcing types to match, when 
                                        -- their underlying representations are equal)

-- we use a (State Bool) monad to carry the 'readingWord' state through all invocations
-- WrappedMonad is used to use the monad as an Applicative Functor
-- This Applicative is then Composed with the actual Count a
wciBody :: Char -> Compose (WrappedMonad (State Bool)) Count a
wciBody c =  coerce (updateState c) where
    updateState :: Char -> Bool -> (Sum Integer, Bool)
    updateState c w = let s = not(isSpace c) in (test (not w && s), s)
    isSpace :: Char -> Bool
    isSpace c = c == ' ' || c == '\n' || c == '\t'

-- using traverse to count words in a String
wci :: String -> Compose (WrappedMonad (State Bool)) Count [a]
wci = traverse wciBody

-- Forming the Product of character counting, line counting and word counting
-- and performing a one go traversal using this Functor product
clwci :: String -> (Product (Product Count Count) (Compose (WrappedMonad (State Bool)) Count)) [a]
clwci = traverse (cciBody <#> lciBody <#> wciBody)

-- the actual wordcount implementation. 
-- for any String a triple of line count, word count, character count is returned
wc :: String -> (Integer, Integer, Integer)
wc str = 
    let raw = clwci str
        cc  = coerce $ pfst (pfst raw)
        lc  = coerce $ psnd (pfst raw)
        wc  = coerce $ evalState (unwrapMonad (getCompose (psnd raw))) False
    in (lc,wc,cc)

-- and then in ghci:
> wc "hello \n world"
(1,2,13)
```

This example has been implemented according to ideas presented in the paper 
[The Essence of the Iterator Pattern](https://www.cs.ox.ac.uk/jeremy.gibbons/publications/iterator.pdf).

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/Iterator.hs)


## Type classes Category, Arrow & Co.
Theses type classes aim at generalizing elements of Monads or Functors.

If you have ideas how these type classes map to specific design patterns please let me know!

# Beyond type class patterns

TBD:
- Chain of Responsibility: ADT + pattern matching the ADT (at least the distpatch variant)

- Currying / Partial application

- Blockchain as Monadic chain of Actions

## Dependency Injection -> Parameter Binding
> [...] Dependency injection is a technique whereby one object (or static method) supplies the dependencies of another object. A dependency is an object that can be used (a service). An injection is the passing of a dependency to a dependent object (a client) that would use it. The service is made part of the client's state. Passing the service to the client, rather than allowing a client to build or find the service, is the fundamental requirement of the pattern.
>
> This fundamental requirement means that using values (services) produced within the class from new or static methods is prohibited. The client should accept values passed in from outside. This allows the client to make acquiring dependencies someone else's problem. 
> (Quoted from [Wikipedia](https://en.wikipedia.org/wiki/Dependency_injection))

In functional languages this is simply achieved by binding a functions formal parameters to values.
See the following example where the function `generatePage :: (String -> Html) -> String -> Html` does not only require a String input but also a rendering function that does the actual conversion from text to Html.
```haskell
data Html = ...

generatePage :: (String -> Html) -> String -> Html
generatePage renderer text = renderer text

htmlRenderer :: String -> Html
htmlRenderer = ...
```
With partial application its even possible to form a closure that incorporates the rendering function:
```haskell
ghci> closure = generatePage htmlRenderer
:type closure
closure :: String -> Html
```



## Adapter -> Function Composition
> "The adapter pattern is a software design pattern (also known as wrapper, an alternative naming shared with the decorator pattern) that allows the interface of an existing class to be used as another interface. It is often used to make existing classes work with others without modifying their source code." 
> (Quoted from https://en.wikipedia.org/wiki/Adapter_pattern)

An example is an adapter that converts the interface of a Document Object Model of an XML document into a tree structure that can be displayed. 

What does an adapter do? It translates a call to the adapter into a call of the adapted backend code. Which may also involve translation of the argument data.

Say we have some `backend` function that we want to provide with an adapter. we assume that `backend` has type `c -> d`:
```haskell
backend :: c -> d
```
Our adapter should be of type `a -> b`:
```haskell
adapter :: a -> b
```

In order to write this adapter we have to write two function. The first is:
```haskell
marshal :: a -> c
```
which translated the input argument of `adapter` into the correct type `c` that can be digested by the backend.
And the second function is:
```haskell
unmarshal :: d -> b
```
which translates the result of the `backend`function into the correct return type of `adapter`.
`adapter` will then look like follows:
```haskell
adapter :: a -> b
adapter = unmarshal . backend . marshal
```
So in essence the Adapter Patterns is just function composition.

Here is a simple example. Say we have a backend that understands only 24 hour arithmetics (eg. 23:50 + 0:20 = 0:10).

But in our frontend we don't want to see this ugly arithmetics and want to be able to add minutes to a time representation in minutes (eg. 100 + 200 = 300).

We solve this by using the above mentioned function composition of `unmarshal . backend . marshal`:

```haskell
-- a 24:00 hour clock representation of time 
newtype WallTime = WallTime (Int, Int) deriving (Show)

-- this is our backend. It can add minutes to a WallTime representation
addMinutesToWallTime :: Int -> WallTime -> WallTime
addMinutesToWallTime x (WallTime (h, m)) =
    let (hAdd, mAdd) = x `quotRem` 60
        hNew = h + hAdd
        mNew = m + mAdd
    in if mNew >= 60
        then
            let (dnew, hnew') = (hNew + 1) `quotRem` 24
            in  WallTime (24*dnew + hnew', mNew-60)
        else WallTime (hNew, mNew)
        
-- this is our time representation in Minutes that we want to use in the frontend
newtype Minute = Minute Int deriving (Show)

-- convert a Minute value into a WallTime representation
marshalMW :: Minute -> WallTime
marshalMW (Minute x) = 
    let (h,m) = x `quotRem` 60
    in WallTime (h `rem` 24, m)

-- convert a WallTime value back to Minutes
unmarshalWM :: WallTime -> Minute
unmarshalWM (WallTime (h,m)) = Minute $ 60 * h + m

-- this is our frontend that add Minutes to a time of a day 
-- measured in minutes
addMinutesAdapter :: Int -> Minute -> Minute
addMinutesAdapter x = unmarshalWM . addMinutesToWallTime x . marshalMW

adapterDemo = do 
    putStrLn "Adapter vs. function composition"
    print $ addMinutesAdapter 100 $ Minute 400
    putStrLn ""
```

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/Adapter.hs)

## Template Method -> type class default functions

> In software engineering, the template method pattern is a behavioral design pattern that defines the program skeleton of an algorithm in an operation, deferring some steps to subclasses. 
> It lets one redefine certain steps of an algorithm without changing the algorithm's structure.
> [Quoted from Wikipedia](https://en.wikipedia.org/wiki/Template_method_pattern)

The TemplateMethod pattern is quite similar to the [StrategyPattern](#strategy---functor). The main difference is the level of granularity. 
In Strategy a complete block of functionality - the Strategy - can be replaced. 
In TemplateMethod the overall layout of an algorithm is predefined and only specific parts of it may be replaced.

In functional programming the answer to this kind of problem is again the usage of higher order functions.

In the following example we come back to the example for the [Adapter](#adapter---function-composition).
The function `addMinutesAdapter` lays out a structure for interfacing to some kind of backend:
1. marshalling the arguments into the backend format
2. apply the backend logic to the marshalled arguments
3. unmarshal the backend result data into the frontend format

```haskell
addMinutesAdapter :: Int -> Minute -> Minute
addMinutesAdapter x = unmarshalWM . addMinutesToWallTime x . marshalMW
```

In this code the backend functionality - `addMinutesToWallTime` - is a hardcoded part of the overall structure.

Let's assume we want to use different kind of backend implementations - for instance a mock replacement. 
In this case we would like to keep the overall structure - the template - and would just make a specific part of it flexible.
This sounds like an ideal candidate for the TemplateMethod pattern:
```haskell
addMinutesTemplate :: (Int -> WallTime -> WallTime) -> Int -> Minute -> Minute
addMinutesTemplate f x =
    unmarshalWM .
    f x .
    marshalMW
```
`addMinutesTemplate` has an additional parameter f of type `(Int -> WallTime -> WallTime)`. This parameter may be bound to `addMinutesToWallTime` or alternative implementations:
```haskell
-- implements linear addition (the normal case) even for values > 1440
linearTimeAdd :: Int -> Minute -> Minute
linearTimeAdd = addMinutesTemplate addMinutesToWallTime

-- implements cyclic addition, respecting a 24 hour (1440 Min) cycle
cyclicTimeAdd :: Int -> Minute -> Minute
cyclicTimeAdd = addMinutesTemplate addMinutesToWallTime'
```

where `addMinutesToWallTime'` implements a silly 24 hour cyclic addition:

```haskell
-- a 24 hour (1440 min) cyclic version of addition: 1400 + 100 = 60
addMinutesToWallTime' :: Int -> WallTime -> WallTime
addMinutesToWallTime' x (WallTime (h, m)) =
    let (hAdd, mAdd) = x `quotRem` 60
        hNew = h + hAdd
        mNew = m + mAdd
    in if mNew >= 60
        then WallTime ((hNew + 1) `rem` 24, mNew-60)
        else WallTime (hNew, mNew)
```

And here is how we use it to do actual computations:
```haskell
templateMethodDemo = do
    putStrLn $ "linear time: " ++ (show $ linearTimeAdd 100 (Minute 1400))
    putStrLn $ "cyclic time: " ++ (show $ cyclicTimeAdd 100 (Minute 1400))
```

### type class minimal implementations as template method

> The template method is used in frameworks, where each implements the invariant parts of a domain's architecture, 
> leaving "placeholders" for customization options. This is an example of inversion of control. 
> [Quoted from Wikipedia](https://en.wikipedia.org/wiki/Template_method_pattern)

The type classes in Haskells base library apply this template approach frequently to reduce the effort for implementing type class instances and to provide a predefined structure with specific 'customization options'. 

As an example let's extend the type `WallTime` by an associative binary operation `addWallTimes` to form an instance of the `Monoid` type class
```haskell
addWallTimes :: WallTime -> WallTime -> WallTime
addWallTimes a@(WallTime (h,m)) b =
    let aMin = h*60 + m
    in  addMinutesToWallTime aMin b

instance Semigroup WallTime where
    (<>)   = addWallTimes
instance Monoid WallTime where
    mempty = WallTime (0,0)
```
Even though we specified only `mempty` and `(<>)` we can now use the functions `mappend :: Monoid a => a -> a -> a` and `mconcat :: Monoid a => [a] -> a` on WallTime instances:

```haskell
templateMethodDemo = do
    let a = WallTime (3,20)
    print $ mappend a a
    print $ mconcat [a,a,a,a,a,a,a,a,a]
```
By looking at the definition of the `Monoid` type class we can see how this 'magic' is made possible:

```haskell
class Semigroup a => Monoid a where
    -- | Identity of 'mappend'
    mempty  :: a

    -- | An associative operation
    mappend :: a -> a -> a
    mappend = (<>)

    -- | Fold a list using the monoid.
    mconcat :: [a] -> a
    mconcat = foldr mappend mempty
```
For `mempty` only a type requirement but no definition is given.
But for `mappend` and `mconcat` default implementations are provided. 
So the Monoid type class definition forms a *template* where the default implementations define the 'invariant parts' of the type class and the part specified by us form the 'customization options'. 

(please note that it's generally possible to override the default implementations)

[Full Sourcecode for this section](https://github.com/thma/LtuPatternFactory/blob/master/src/TemplateMethod.hs)

## TBD: Factory -> Function Currying

## Conclusions
> Design patterns are reusable abstractions in object-oriented software. 
> However, using current mainstream programming languages, these elements can only be expressed extra-linguistically: as prose,pictures, and prototypes. 
> We believe that this is not inherent in the patterns themselves, but evidence of a lack of expressivity in the languages of today. 
> We expect that, in the languages of the future, the code parts of design patterns will be expressible as reusable library components. 
> Indeed, we claim that the languages of tomorrow will suffice; the future is not far away. All that is needed, in addition to commonly-available features, 
> are higher-order and datatype-generic constructs; 
> these features are already or nearly available now.  
> Quoted from [Design Patterns as Higher-Order Datatype-Generic Programs](http://www.cs.ox.ac.uk/jeremy.gibbons/publications/hodgp.pdf)


<!--
## TBD: Conclusion
> While we (me included) have been on an a thirty-odd year long detour around object-orientation, I don't think all is lost. 
> [Quoted from blog.ploeh.dk](http://blog.ploeh.dk/2018/03/05/some-design-patterns-as-universal-abstractions/)

>  In the functional-programming world, traditional design patterns generally manifest in one of three ways:
> - The pattern is absorbed by the language.
> - The pattern solution still exists in the functional paradigm, but the implementation details differ.
> - The solution is implemented using capabilities other languages or paradigms lack. (For example, many solutions that use metaprogramming are clean and elegant — and they're not possible in Java.)
>
> [Quoted from IBM developerworks](https://www.ibm.com/developerworks/library/j-ft10/index.html)

http://blog.ploeh.dk/2018/03/05/some-design-patterns-as-universal-abstractions/
http://blog.ploeh.dk/2017/10/04/from-design-patterns-to-category-theory/
-->

# some interesting links
https://www.ibm.com/developerworks/library/j-ft10/index.html

http://blog.ezyang.com/2010/05/design-patterns-in-haskel/

https://staticallytyped.wordpress.com/2013/03/09/gang-of-four-patterns-with-type-classes-and-implicits-in-scala/

http://norvig.com/design-patterns/design-patterns.pdf

[Scala Typeclassopedia](https://github.com/tel/scala-typeclassopedia)
