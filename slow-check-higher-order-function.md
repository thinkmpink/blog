# A Slow Check for a Higher-Order Function

As a relative newcomer to the Haskell community, I'm just beginning to acquaint myself with the mechanisms Haskellers use to verify their code. I really like the idea of property-based checks. I can write an algorithm and test each part of it by affirming its properties, either during or after I compile my code.

### First option: QuickCheck

To me, [QuickCheck](http://hackage.haskell.org/package/QuickCheck) sets the lowest barrier to entry. It can randomly generate many unit tests based on properties I express. There appears to be a nice set of properties I can use in the package `[Test.Invariant](http://hackage.haskell.org/package/test-invariant-0.4.5.0/docs/Test-Invariant.html)`, including testing whether my function is idempotent (a.k.a. running it more than once has no greater effect than running it once), or monotonic increasing, and so on.

### Second option: Type-level programming

Type-level programming offers a tantalizing promise, on the other hand, of guaranteeing properties **at compile time**! If it works, this is a much stronger guarantee of the correctness of my code than a unit test. Randomized unit tests represent a high probability of correctness, whereas compilation represents proof and certainty.

I began to dig into type-level programming when I wanted to write a web server with [Servant](http://hackage.haskell.org/package/servant). Realizing I didn't recognize the syntax of the Servant's types (what are those apostrophes doing everywhere?), I backed up to the basics. Friend and fellow Recurser [James Vaughan](https://github.com/vaughanj10) directed me to Sandy Maguire's book, [Thinking With Types](https://thinkingwithtypes.com/), and we began the book's exercises.

Here's the cool bit: because of a profound correspondence in the Universe, category theory, logic, and the types we use in programs [are in some sense equivalent](https://wiki.haskell.org/Curry-Howard-Lambek_correspondence). In other words, we can prove a proposition by writing a function (that typechecks)!

So I got all excited about my curry' function in the first exercise (spoiler alert!) and its corresponding uncurry' function:

```haskell
curry' :: ((b, c) -> a) -> b -> c -> a
curry' f b c = f (b, c)

uncurry' :: (b -> c -> a) -> (b, c) -> a
uncurry' f (b, c) = f b c
```

### First attempt: Equational Reasoning

When the book asked me to show that `curry' . uncurry' == id`, I first thought,
"Okay, I will be a Good Boy and use [Equational Reasoning](http://www.haskellforall.com/2013/12/equational-reasoning.html)." Since the `=` function in Haskell means the same as `=` in math, unlike most programming languages where it means "Watch out, everything is about to become more complicated", I thought I was doing the right thing.

To my dismay, my reasoning didn't compile. It failed on the premise that the same function had multiple definitions. While I can understand this failure--as  though the compiler is asking me "Which implementation do you want?"--it would be nice if the admittedly super-powered GHC could pick the most efficient definition for a function when I write multiple equivalent definitions.

I set that approach aside for the moment, and decided to try my hand at QuickCheck.

### Second attempt:

According to the blog posts and online documentation, I believed, testing the proposition that `curry' . uncurry' == id` could be achieved by plugging that statement into [HSpec](NEED LINK!)'s DSL for executable specifications:

```haskell
main = hspec $ do
  describe "uncurry\'" $ do
    it "inverts curry\'" $
      property $ curry' . uncurry' == id
```

That would be awesome! Unfortunately, it doesn't typecheck. Can you see why?

First of all, if you load it into GHCi, you will see that

```
$ ghci
...
Prelude> :t curry' . uncurry'
curry' . uncurry' :: (a -> b -> c) -> a -> b -> c
```

whereas

```
Prelude> :t id
id :: a -> a
```

In other words, the types of `curry' . uncurry'` and `id` don't line up. Seeing the misalignment, I tried to provide an equivalent (but less elegant) statement of the property. It would have more of the argument types filled in, to meet the demands of QuickCheck's inferencing capabilities. This didn't work; I kept getting the same compilation error.

Back on the blog posts, I realized QuickCheck might need concrete types to work. In other words, instead of defining a property over abstract, unconstrained types like `(a -> b -> c)`, maybe I would need to define the same property over `(Int -> Int -> Int)`. No dice.

After some searching, I learned that I needed a little QuickCheck boilerplate in order to support the arbitrary generation of functions, in the form of the `Fun` data type. The catch here is for multi-parameter higher-order functions. Say I need to generate a random function of type `f :: a -> b -> c`. My QuickCheck property type signature will look like:

```haskell
prop_myHigherOrderProp :: Fun (a, b) c -> ...
```

The arguments to the `Fun` type constructor are represented in curried form. Notice, however, that `Fun (a, b) c` could be used to describe a higher-order random function that takes an `a` and then a `b` and returns a `c`, or a function that takes a pair `(a, b)` and returns a `c`. Either of the following definitions for `prop_myHigherOrderProp` is viable:

```haskell
prop_myHigherOrderProp (Fn f) (a, b) = ...
```

and

```haskell
prop_myHigherOrderProp (Fn2 f) a b = ...
```

Once over that hurdle, I finally got to a working definition for my property:

prop_curryAfterUncurryIsId :: Fun (Int, Int) Int -> Int -> Int -> Bool
prop_curryAfterUncurryIsId (Fn2 f) a b = curry' (uncurry' f) a b == f a b

I'm not thrilled that `id` is nowhere to be found in the definition, but I think the meaning comes through if you are comfortable with the syntax. Needless to say, this was all harder to verify than I hoped, but for now I'm chalking it up to my inexperience using QuickCheck.

If you liked this post or have comments to share, please reach out to me! What experiences have you had using QuickCheck?
