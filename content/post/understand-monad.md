---
date: 2018-10-13
title: Understand Monad in Functional Programming
tags: ["FP", "Scala"]
categories: ["Functional Programming"]
---

## Why Functional Programming?

The facination of doing programming is to solve the exist problems. Revisit the history of software development, at first place we have **procedure-oriented programming**. As the system becomes bigger, the codebase is so hard to maintain and adding new features. Then a few genius comes out the idea with **OOP** with the **SOLID** principles to guide the daily dev work.

If there is one rule in software development, i think that must be `no silver bullet`. The OOP has ruled the system development for more that 20 years (The number comes from JAVA becomes popularity). More and more devs find object-oriented programming sucks:

- Mutable state of instances, variables
- Hard to conform with SOLID when business logic becomes messy
- Hard to test caused by improper DI
- So many lines of code
- Boring ...

Now people are thinking how to solve them:

* There is no mutable variables
* The functionality should be encapsulated
* I don't want exception, the compute unit (AKA function) should be pure
* The code should be clean
* Easy to test
* Parallelism
* Make me happy ...

That's what the FP try to implemented. What's the design principle for FP? Category theory. It's simple as its name. As the saying goes, there is no silver bullet. 

What sucks in FP?

* Learning curve
* Too young
* The world is not perfect
* *Please add more* ...

## Pure Function

A function can be pure or impure. Usually we use pure function in FP as much as we can. There are a few conditions for pure function:

1. Give the same input argument to the function, it should always return same output.
2. The function should only depends on the return value to change the whole "world", which means no side effects. The side effects can be an exception, mutate an object, IO operation, etc. 

I will give one example to understand pure function:

```scala
// pure function
def add(i: Int, y: Int) =
  i + y

// impure function
def addAndPrint(i: Int, y: Int) = {
  val result = i + y
  println(s"The result is ${result}")
  result
}
```

A short summary is:

* Output depends only on input.
* No side effects

With pure function, it's possible to make sure our program is **referential transparency** which means that an expression always evaluate to the same result in any piece of code. Referential transparency makes the code easier to reason about.

What can be done if we don't read any inputs and write any outputs?

Usually, we can refactor impure function to be pure by adding a proper **context** (here a context is what a value wrapped in.) like **Option**, **cats.effect.IO** or **monix.eval.Task** to the function return type. Some cases are:

* Use **Either** to represent exception
* Use **IO** or **Task** to delay the I/O operation

## Function Composition

Basicly, FP is about fucntion composition. That means:

```scala
// A, B, C are types
f: A -> B, 
g: B -> C, 
g compose f: A -> C
```

But how should we deal with the values wrapped in context like Option or IO? There could be lots of scheleton code if we still use simple function composition in this case. Then, there comes **monad**, it opens the context and keeps the context same in the following computation. Usually we use for comprehension to simplise the control flow:

```scala
val result = for {
  a <- Option.apply(1) // this is equal to Some(1)
  b <- Option.apply(2)
} yield a + b

// result is: Some(3)
```



## Monad

### Define a Monad

In this section, we will implement an 'ugly' Option monad, without considering any variance in type marameter:

```scala
sealed trait Option[+A] {
  def flatMap[B](func: A => Option[B]): Option[B] =
    this match {
      case None => None
      case Some(a) => func(a)
    }
  
  def map[B](func: A => B): Option[B] =
    this match {
      case None => None
      case Some(a) => Some(func(a))
    }
}
case object None extends Option[Nothing]
case class Some[T](value: T) extends Option[T]
```

The usage of this either monad:

```scala
val finalEither = for {
    a <- Right[String, Int](1)
    b <- Right[String, Int](2)
  } yield (a + b)
```

### Monad Laws

When defining monad, it should follow a few laws, these laws maintain the basics of a category.

Identity law:

```scala
def unit[A](a: A): F[A]
def f[A](a: A): F[A]

// With same a, the following equation should be met.
f(a).flatMap(unit) == f(a)
unit(a).flatMap(f) == f(a)
```

Associative law:

```scala
// x is a F[A]
x.flatMap(f).flatMap(g) == x.flatMap(a => f(a).flatMap(g))
```



## Free Monad

### Why need it

There is developers want to decomose data from the interpreter, which means there can be multiple interpretation for the same piece of computation. It would be helpful if we want to invoke different computation in different place. In this case, we need to wrap such computation in a context which can be composed with the computation in same context. This is a special monad which is called free. Typically ADT is common used to represent the data.

### How to use

In practice, we can view **Free** as a clever way of forming **Monad** with providing **Functor**.

Following steps show the way to use free monad provides by cats library (need to add `cats-free` dependency): 

1\. Create custome ADT to represent the operation:

   ````scala
   sealed trait KVStoreA[A]
   
   case class Put[T](key: String, value: T) extends KVStoreA[Unit]
   case class Get[T](key: String) extends KVStoreA[Option[T]]
   case class Delete(key: String) extends KVStoreA[Unit]
   ````

2\. Define free monad with Free for above ADT:

   ```scala
   import cats.free.Free
   
   // We haven't define Functor, but use CoYoneda[F, _] is functor for any F.
   type KVStore[A] = Free[KVStoreA, A]
   ```

3\. Using **liftF** Create DSL related functions which return above free monad:

   ```scala
   import cats.free.Free.liftF
      
   // Put returns nothing (i.e. Unit).
   def put[T](key: String, value: T): KVStore[Unit] =
     liftF[KVStoreA, Unit](Put[T](key, value))
      
   // Get returns a T value.
   def get[T](key: String): KVStore[Option[T]] =
     liftF[KVStoreA, Option[T]](Get[T](key))
      
   // Delete returns nothing (i.e. Unit).
   def delete(key: String): KVStore[Unit] =
     liftF(Delete(key))
      
   // Update composes get and set, and returns nothing.
   def update[T](key: String, f: T => T): KVStore[Unit] =
     for {
       vMaybe <- get[T](key)
        _ <- vMaybe.map(v => put[T](key, f(v))).getOrElse(Free.pure(()))
     } yield ()
   ```

4\. Build a program with constructed function:

   ```scala
   def program: KVStore[Option[Int]] =
     for {
       _ <- put("cats", 2)
       _ <- update[Int]("cats", (_ + 12))
       n <- get[Int]("cats")
       _ <- delete("cats")
     } yield n
   ```

5\. Implement the interpreter which is a natural transformation to interpret each operation:

   ```scala
   import cats.{Id, ~>}
   import scala.collection.mutable
   
   def impureCompiler: KVStoreA ~> Id  =
     new (KVStoreA ~> Id) {
   
       // a very simple (and imprecise) key-value store
       val kvs = mutable.Map.empty[String, Any]
   
       def apply[A](fa: KVStoreA[A]): Id[A] =
         fa match {
           case Put(key, value) =>
             println(s"put($key, $value)")
             kvs(key) = value
             ()
           case Get(key) =>
             println(s"get($key)")
             kvs.get(key).map(_.asInstanceOf[A])
           case Delete(key) =>
             println(s"delete($key)")
             kvs.remove(key)
             ()
         }
     }
   ```

6\. Run the program with defined interpreter:

   ```scala
   // Free[_] is just a recursive structure, which similar to List.
   val result: Option[Int] = program.foldMap(impureCompiler)
   ```

### What's the implementation

The implementation of free monad, details go to this [post](https://underscore.io/blog/posts/2015/04/23/deriving-the-free-monad.html):

```scala
sealed trait Free[F[_], A] {
  def flatMap[B](f: A => Free[F, B])(implicit functor: Functor[F]): Free[F, B] =
    this match {
      case Return(a)  => f(a)
      case Suspend(s) => Suspend(s map (_ flatMap f))
    }
}

final case class Return[F[_], A](a: A) extends Free[F, A]
final case class Suspend[F[_], A](s: F[Free[F, A]]) extends Free[F, A]

object Free {
  def point[F[_]](a: A): Free[F, A] = Return[F, A](a)
}
```

Usually, we will have a lot of ADTs in the code, but unrelated monad do not compose. Cats has given an option to chain the different ADT by using **coproduct** type. We are not going to discuss the detail here, but worth to dig it more [here](https://www.47deg.com/blog/fp-for-the-average-joe-part3-free-monads/) if you are interested.



## Monad transformer

### Why need it

Imagine there is a piece of code to fetch the address of a user by its id:

```scala
def findUserById(id: Long): Future[User] = ???
def findAddressByUser(user: User): Future[Address] = ???
```

We can use for-comprehension to control the workflow since they share the same context Future and Future has a build-in `flatMap`.

```scala
def findAddressByUserId(id: Long): Future[Address] =
  for {
    user    <- findUserById(id)
    address <- findAddressByUser(user)
  } yield address
```

There is a situation that we can't find a user if the id is not exist, event the address can be optional for a user.

```scala
def findUserById(id: Long): Future[Option[User]] = ???
def findAddressByUser(user: User): Future[Option[Address]] = ???
```

Then the workflow need to be changed:

```scala
def findAddressByUserId(id: Long): Future[Option[Address]] =
  findUserById(id).flatMap {
    case Some(user) => findAddressByUser(user)
    case None       => Future.successful(None)
  }
```

It's working but not as beautiful as previous. Now we comes to monad transformers.

### How to use

Use **OptionT** provided by cats in for-comprehension:

```scala
def findAddressByUserIdOptionT(id: Long): OptionT[Future, Address] =
  for {
    user <- OptionT(findUserById(id))
    address <- OptionT(findAddressByUser(user))
  } yield address
```

### What's the implementation

The brief definition of OptionT:

```scala
case class OptionT[F[_], A](value: F[Option[A]]) {
  def map[B](f: A => B)(implicit F: Functor[F]): OptionT[F, B] =
    OptionT(value.map(_.map(f)))

  def flatMap[B](f: A => OptionT[F, B])(implicit F: Monad[F]): OptionT[F, B] =
    OptionT(value.flatMap(
      _.fold(Option.empty[B].pure[F])(
        f andThen ((fa: OptionT[F, B]) => fa.value))))
}
object OptionT {
  def pure[F[_]: Applicative, A](a: A): OptionT[F, A] =
    OptionT(a.pure[Option].pure[F])
}
```

One thing need to mention is that many monad transformers could be stack unsafe, like `StateT` .



## Extensible Effect

### Why need it

Monad transformers has a limited nest layers, also must keep the order same in different computation. What if we have some super way to rule all the monads? In this case we can walk through the control flow without worring the different monad. Finally it will free us from assembling different manads.

The principle ideas to understand effects are:

>* An effect is most easily understood as an interaction between a sub-expression and a central authority that administers the global resources of a program.
> * An effect can be viewed as a message to the central authority plus enough information to resume the suspended calculation.

### How to use

```scala
type Stack = Fx.fx2[Option, StringEither]

type HasOption[R] = MemberIn[Option, R]
type StringEither[A] = Either[String, A]
type HasEither[R] = MemberIn[StringEither, R]

def program[R: HasOption: HasEither] = for {
  a <- fromOption(Some(1))
  b <- fromEither(Right(2))
} yield a + b

println(program[Stack])
println(program[Stack].asInstanceOf[ImpureAp[_, _, _]].continuation.functions)

val result = program[Stack].runOption.runEither.run // Right(Some(3))
```

### What's the implementation

The core of Eff library is Eff monad and the open union. Defining effects and their interactions is done by the user. There are some common effects provided by the library like ReaderEffect, IOEffect etc for convenience.



## Reference

* [Free and Freer Monads: Putting Monads Back into Closet](http://okmij.org/ftp/Computation/free-monad.html)
* [Pure Functions and I/O](https://alvinalexander.com/scala/fp-book/pure-functions-and-io-input-output)
* [There Can Be Only One...IO Monad](http://degoes.net/articles/only-one-io)
* [Monix’s Task vs cats.effect.IO](https://monix.io/blog/2018/03/20/monix-vs-cats-effect.html)
* [Monadic IO: Laziness Makes You Free](https://underscore.io/blog/posts/2015/04/28/monadic-io-laziness-makes-you-free.html)
* [There Can Be Only One...IO Monad](http://degoes.net/articles/only-one-io)
* [Where does the term “Monad” come from?](https://english.stackexchange.com/questions/30654/where-does-the-term-monad-come-from)
* [Monad (functional programming) in Wikipedia](https://en.wikipedia.org/wiki/Monad_(functional_programming))
* [Free Monad in Cats](https://typelevel.org/cats/datatypes/freemonad.html)
* [Deriving the Free Monad](https://underscore.io/blog/posts/2015/04/23/deriving-the-free-monad.html)
* [Free Monads and the Yoneda Lemma](http://blog.higher-order.com/blog/2013/11/01/free-and-yoneda/)
* [Purely Functional I/O in Scala](http://blog.higher-order.com/assets/scalaio.pdf)
* [Stackless Scala With Free Monads](http://blog.higher-order.com/assets/trampolines.pdf)
* [Monad Transformers for the working programmer](https://blog.buildo.io/monad-transformers-for-the-working-programmer-aa7e981190e7)
* [Cats: OptionT](https://typelevel.org/cats/datatypes/optiont.html)
* [No More Transformers: High-Performance Effects in Scalaz 8](http://degoes.net/articles/effects-without-transformers)
* [Why free monads matter](http://www.haskellforall.com/2012/06/you-could-have-invented-free-monads.html)
* [Referential transparency](https://wiki.haskell.org/Referential_transparency)

