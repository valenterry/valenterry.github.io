---
layout: post
title: Master Scalas building blocks
---

# Master Scalas building blocks

Scala is a language with a bunch of powerful features. These can be used as building blocks for higher level concepts.
They can, however, also be used in ways that makes code more unsafe in terms of runtime errors and harder to read and extend.

As a professional Scala developer for many years I'd like to share my perspectives about how Scalas features are best used.
This advice is especially directed to beginner and intermediate level Scala developers.
It is partly opinionated towards (pure) functional programming and might not be apply when performance or compatibility are of high relevance.
Sometimes they have to be used for creating easy writable DSLs, too.

In most cases though it is a good point to start with when in doubt of how to use Scala for developing and maintaining larger programs.  

We will start with the most common building blocks in Scala and how to use and especially not to use them.

## Implicits

Implicits are a rather unique feature of Scala, not found in most languages.
They are very powerful and thus can lead to very irritating behaviour and hard to read code when used wrong.
On the other hand they allow a few patterns that are overall safe to use and benefit productivity and extensibility.

### Implicit defs
Implicit methods should generally be avoided, especially for automatic conversions.
There are exceptions:
- Creating a new instance of a custom defined class to extend a class with new functionality. In this case the implicit class pattern can and should be used instead.
- Typelevel programming. That is, the method must have only implicit parameters.
 
Examples:
Bad:
{% highlight scala %}
implicit def(s: String): List[Char] = ???
{% endhighlight %}

Good:
{% highlight scala %}
implicit def (s: String): StringOps = new StringOps(s)
{% endhighlight %}

Better:
{% highlight scala %}
implicit case class StringOps(s: String) { … }
{% endhighlight %}

Also good:
{% highlight scala %}
implicit def orderingByNumeric[A](implicit ord: Numeric[A]): Ordering[B] = ???
{% endhighlight %}

### Implicit method parameters
Implicit method parameters should only be used for passing type class instances and not for passing contexts/configurations.
Making (parts of) arguments implicit because it is tedious to pass them down the call chain is a code smell and there usually exists a way around, e.g. by using the Reader Monad or creating a DSL with the Free Monad.
Providing an implicit argument by programmers mistake must be extremely rare/hard and thus: the more general the implicit parameter the worse. 

Examples:
- BAD: def greetUser(name: UserName)(implicit lang: Language) = ???
- WORSE: def allUsers(repository: UserRepository)(implicit ec: ExecutionContext) = ???
- DOOMED: def getAllUsers(repository: UserRepository)(implicit ec: ExecutionContext, timeoutLimit: Long) = ???
- GOOD: def serialize[A](obj: A)(implicit si: SerializeInstructions[A]) = ???
- BETTER: def serialize[A: SerializeInstructions](obj: A) = ???

## Traits
### Sumtypes / Uniontypes
One recommended use case for traits is the encoding of sumtypes. A sumtype can be represented like that:
sealed trait Shape
	final case class  Circle(radius: Double) extends Shape
	final case class  Rectangle(width: Double, height: Double) extends Shape
	final case class  Cone(radius: Double, height: Double) extends Shape
	final case object Rheinturm extends Shape
	final case object KoelnerDom extends Shape
	
We now have safe pattern matching (compiler warns us if we forget a case) and an easy to understand representation of the different kind of things that are handled by our application.
Don't forget to add the sealed keyword! This is mandatory. Never leave that out. Also note that the trait should not have methods defined (also see [Inheritance](#Inheritance of traits)).

Advanced: What if you don't have control over some or all of the classes (Circle, Rectangle, Cone, ...) but still want to have type-safe transform functionality similiar to pattern matching?
Answer: Use shapeless Coproduct: type Shape = Circle :+: Rectangle :+: Some3rdPartyShape :+: Another3rdPartyShape :+: CNil

### Inheritance of traits
Other than in Java, interface/trait inheritance to provide a common set of functionality is usually discouraged in Scala.
It is, however, okay to use inheritance under the condition that you are in complete control of the code. That means, you must be able, at any time, to rewrite every single part of any code that could possibly use your trait (including 3rd party code) and you have weighted the benefits vs. costs in comparison with typeclasses.
That means, using traits when writing a library for others is an absolute NOGO. Don't do this.

- Inheritance is okay for ADTs and also in cases like this:
sealed trait LogMessage {
	def msg: String
}
case class InfoMessage(msg: String) extends LogMessage
case class WarnMessage(msg: String) extends LogMessage
case class ErrorMessage(msg: String) extends LogMessage

trait Drawable {
	def asImage: JPG
}
case class Circle(...) extends Drawable {
	override def asImage: JPG = ???
}

Note, however, that typeclasses should be used as soon as the Drawable trait gets more complex or used rather often.

### Abstract classes vs. traits
Abstract classes are similiar to traits. Rule of thumb: always use trait unless you have specific reasons for an abstract class.
In some situations it is convenient to have constructor parameters in ADTs for types with fixed values and in such cases traits add boilerplate.
This is an example of an okayish usage of abstract classes:

sealed abstract class ServiceError(msg: String)
case object Timeout extends ServiceError("The Request to the Service took too long")
case object InvalidResponse extends ServiceError("Could not parse the Service response")

Using a sealed trait it would look like that:
sealed trait ServiceError {
  def msg: String
}
case object Timeout extends ServiceError {
  override val msg: String = "The Request to the Service took too long"
}
case object InvalidResponse extends ServiceError {
  override val msg: String = "Could not parse the Service response"
}

An important difference between abstract classes and traits is multiple inheritance which is possible with traits but not with abstract classes.
This does not matter though, because we discourage multiple inheritance (and often even inheritance itself) anyways most of the time.


### Typeclasses

### Cakepattern





### Usage of traits for library writers
When writing a library that provides a method that accepts objects by the client (thus not under your control), never force inheritance usage.
That is, don't make the client inherit from a trait, abstract class or anything else. Instead, always use typeclasses when the library needs certain functionality that should be provided by its client.

Examples:

- LoggingLibrary BAD:
trait StringSerializeable {
    def serialized: String
}
def log[A <: StringSerializeable](obj: A) = writeToLogFile( obj.serialized )
 
- LoggingLibrary GOOD:
trait StringSerializer {
    def serialize[A](obj: A): String
}
def log[A: StringSerializer](obj: A) = writeToLogFile( implicitly[StringSerializer].serialize(obj) )

## Classes
### Case classes
case classes are „java-DTOs“. They must never contain state (vars) or interact with other components in any way.
Also, even if technically possible, they must not be extended. It is boilerplaty but recommended to finalize every case class.
The only thing that is allowed are „convenience“ methods that use only data from inside of the case class (and no additional data).
The reason here is, that Plain-Data classes should only ever be coupled with classes that they existentially depend on.
In the example: Urls can exists in a world without anything like users, so don't couple Urls to the fact that there exists a thing like a user.

Examples:
final case class Url(host: String, port: Int) {
    def urlString = s"$host:$port" //Okay!
    def isHttps = host.take(5) == "https" //"Okay"!
    
    //Returns http(s)://${user.name}@${user.password}:$password@$host:$uri
    def withUserCredentials(user: User) = ??? //Bad! Avoid this. Use implicit class instead
}

### Constructing classes with constraints
Sometimes a class must adhere to certain constraints. These constraints should be represented as types as much as possible and reasonable.
When enforcing the constraints at compiletime is not possible or reasonable, don't (only) make the constructor throw exceptions.
Instead, provide a factory-function in the companion-object that tries to construct the class and does not simply return the class itself but a type that indicates if the construction was successfull or not.

Examples:
- Not very nice:
final case class PositiveInt(value: Int) {
    if(value < 0) throw new IllegalArgumentException("PositiveInt must have positive value")
}
- Somehow better:
final case class PositiveInt private(value: Int) {
    throw new IllegalArgumentException("PositiveInt must have positive value")
}
object PositiveInt {
    type Error = String
    def fromInt(value: Int): Either[Error, PositiveInt] =
        if(value >= 0)  Right(PositiveInt(value))
        else            Left("PositiveInt must have positive value")
}
- But:
    - PositiveInt(-50) still compiles
    - .copy(value = -50) still compiles
    - Making PositiveInt a normal class throws away all convenience (pattern matching, hashcode/equals, copy, ...)

- Crazy workaround:
object Numbers {
  type Error = String
  sealed abstract case class PositiveInt private[Numbers](value: Int) {
    require(value >= 0)
  }
  object PositiveInt {
    def fromInt(value: Int): Either[Error, PositiveInt] =
      if (n >= 0) Right(new PositiveInt(value){})
      else Left("PositiveInt must have positive value")
  }
}

PositiveInt.fromInt(42) //Some(PositiveInt(42))
PositiveInt.fromInt(-42) //None
new PositiveInt(-42) //Compile error
PositiveInt.fromInt(42).copy(value = -42) //Compile error
PositiveInt.fromInt(42) match { //Compiles and works!
	case PositiveInt(value) => ??? 
}

Summarized: choose you the least bad of them...

If you rather want to rely on macros for some cases, use the "refined" library (https://github.com/fthomas/refined)


## Pattern matching
Pattern matching is a powerful language feature and makes a lot of things easier. Because of its power it also comes with some dangers.
For that reason, pattern matching should only used when needed and there are some basic rules to be followed to make its usage safe.

### Simple comparisons
Pattern matching can be used to check conditions and to extract values. If only simple conditions are checked, use if/else instead of pattern matching.
Example:
birthyear match {
	case 1989 => println("this is the best year")
	case 2017 => println("You are to young to read this anways")
}
Instead use
if(birthyear == 1989) println("this is the best year")
else if(birthyear == 2017) println("You are to young to read this anways")

The reason is that pattern matching should ideally indicate that all cases are handled which is not the case here. Thus, if you don't need the power of pattern matching, don't use it.
It's not per se better than using if/else. One advantage of if/else is, that one can immediatly see that there is no else clause, thus some cases will most likely not be caught.
 
If you need the power of pattern matching e.g. because you wan't to extract values from case classes and match on structure, take care to make pattern matching safe.

### Case classes of sealed traits
Pattern matching is safe if you match on case classes of sealed traits without using guards. This rule recursively applies for matching things within case classes.
For the built in primives like String, Boolean, Int, ... it is also safe to match on their values even though they are not case classes because the compiler will warn you regardless.
If you match on non-case classes or use guards, you must always provide a default case that applies when all other matches failed.
- Examples

This one is okay because the compiler will warn (or fail to compile with compiler flag)
"hello" match { case "really hello" => ??? }

case class Seed()

sealed trait Fruit
case class Grape(seedWeight: Option[Double]) extends Fruit
case class Banana(length: Float) extends Fruit
case class Apple(seeds: List[Seed]) extends Fruit

This one is also good. Missing classes (like missing Apple in this example) will create compiler warnings/errors
someFruit match {
	case Grape(_) => ???
	case Banana(length) => ???
}

Even the next one is fine, because messing up with their inner values will throw warnings/errors as long as they are primitives
someFruit match {
	case Grape(_) => ???
	case Banana(20.05) => ???
	case Apple(_) => ???
}
Will result in compiler saying: Warning: match may not be exhaustive. It would fail on the following input: Banana((x: Float forSome x not in 20.05))

Now a guard is used. The guard checks if the length is over 20. This will NOT trigger compiler warnings/errors and thus can lead to unsafe pattern matchings, resulting in MatchErrors
someFruit match {
	case Grape(_) => ???
	case Banana(length) of length > 20 => ???
	case Apple(_) => ???
}
Therefore, don't do this! Either check the conition behind the case expression (which forces you to deal with both cases unless you use impure side effects) or add a default case at the end.

This one is fine:
someFruit match {
	case Banana(length) of length > 20 => ???
	case _ => ???
}

The following matches if the Fruit is a Grape and then if it has a seed to extract the weight from. This get warnings/errors because we don't match for Grapes without a seed.
Adding a ```case Grape(None) => ???``` would make it compile again.
someFruit match {
	case Grape(Some(weight)) => ???
	case Banana(_) => ???
	case Apple(_) => ???
}

However, consider the following example:
someFruit match {
	case Grape(_) => ???
	case Banana(_) => ???
	case Apple(List(Seed(), Seed(), Seed())) => ???
}
We only match for Apples with 3 seeds and obviously miss some cases which can result in a MatchError.
However, we might expect the compiler to tell us that we forgot some cases, but it doesn't.
The reason is that ```List``` is not a case class!
In Scala, Lists are constructed recursively using the ```::``` case class. Thus, if we use ```::``` to match on the seeds, the compiler will help us out!
someFruit match {
	case Grape(_) => ???
	case Banana(_) => ???
	case Apple(Seed() :: Seed() :: Seed() :: Nil) => ???
}
This will result in: Warning: match may not be exhaustive. It would fail on the following inputs: Apple(List(_)), Apple(List(_, _)), Apple(List(_, _, _, _)), Apple(Nil)

The rule of thumb is therefore: Try to only match on sealed traits of case classes on every level of your pattern matching.
Try to avoid pattern matching on non-sealed traits, non-case classes or if you use guards.
But if you do, always provide a default case. 