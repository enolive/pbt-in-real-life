---
# You can also start simply with 'default'
theme: enolive
# some information about your slides (markdown enabled)
title: Property-Based Testing IRL
info: |
  ## Property-Based Testing IRL
  ...not only for mathematicians!
transition: slide-left
mdc: true
# open graph
seoMeta:
  ogImage: auto
src: ./intro.md
---

---
src: ./about.md
---

---
layout: cover
---

# Motivation

---
layout: two-cols-header
class: fade
---

# Example Testing

::left::

```kotlin [concrete example]
it("generates fizz") {
  val result = fizzBuzz(3)
  
  result shouldBe "Fizz"
}
```

::right::

<v-clicks>

- easier to understand
- many tests necessary
- lots of duplication
- requires lots of testing discipline
- you will most probably miss edge cases
- duplication solvable by parameterizing the test

</v-clicks>

---
layout: two-cols-header
class: fade
---

# Property-Based Testing

::left::

```kotlin [property] {all|3,7|2|all}
it("numbers divisible by 3 produce a Fizz") {
  val numbersDivisibleBy3 = Arb.int().filter { it % 3 == 0 }
  checkAll(numbersDivisibleBy3) { input ->
    val result = fizzBuzz(input)
  
    result shouldContain "Fizz"
  }
}
```

::right::

<v-clicks>

- harder to understand
- property is something all/many cases share
- the hard part is to come up with good properties!
- ... without duplicating the implementation 😉

</v-clicks>

---
layout: image-right
image: /research3.png
class: fade
---

# More properties

<v-clicks>

- numbers divisible by 5 produce Buzz
- result is never empty
- pattern repeats every 15 numbers
- every sequence of 3 numbers contains a Fizz
- function works for any possible number

</v-clicks>

---
layout: image-right
image: /stop.png
class: fade
---

# What is Property-Based Testing **not**?

<v-clicks>

- replacement for Example Testing
- exhaustive
- proof of code correctness
- only for mathematicians
- only for complicated algorithms

</v-clicks>

---
layout: fact
---

- Tests are **hypotheses** about the correctness of the code.
- Therefore, they **cannot be proven**, only **falsified**.
- Finding a good, **falsifiable** hypothesis is the hard thing!

---
layout: cover
---

# How it works

---
class: text-3xl fade
---

<v-clicks>

- each property executed many times with random-ish data
- on failure
- display used input
- try to find a simpler example, via [Shrinking](https://kotest.io/docs/proptest/property-test-shrinking.html)
- display deterministic seed for reproduction

</v-clicks>

---

# Can you spot the error?

```kotlin {all|2,4}
it("upper cased version of any string has the same length") {
  val evilChars = ...
  val string = Arb.string(
    codepoints = Codepoint.alphanumeric().merge(evilChars)
  )
  checkAll(string) { input ->
    val upperCased = input.uppercase()

    upperCased.shouldHaveSameLengthAs(input)
  }
}
```
---

# Error Output

```log
Property failed after 1 attempts
java.lang.AssertionError: Property failed after 1 attempts

	Arg 0: "ß" (shrunk from äßeäÜjEnÜcäküäseÖÜäiÖÜGzÜOocög1ßÜßhlßüDXäÜ4e)

Repeat this test by using seed -2513300078753967404

Caused by: "SS" should have the same length as "ß"
```

<v-click>

- Test can be configured to use this seed
- Some frameworks automate that on test failure (jqwik, kotest)

</v-click>

<style>
ul {
  @apply mt-8 text-2xl;
}
</style>

---
layout: cover
---

# Working with Arbitraries

---
layout: image-right
image: /arbitraries.png
class: fade
---

# What is an Arbitrary?

<v-clicks>

- Random Generators with shrinking
- Biased towards edge cases
- Arbitraries for primitive types available in the frameworks
- Combinable to something more complex

</v-clicks>

---

# Primitives

```kotlin {1-5|all}
val constant = Arb.constant(42)

val arbNumber = Arb.int()

val arbNumberPositive = Arb.int(min = 0)

val arbStatus = Arb.enum<Status>()
// to my knowledge, kotest is the only PBT framework having exhaustive generators
val exStatus = Exhaustive.enum<Status>()

val arbString = Arb.string()

val arbHebrewString = Arb.string(
  range = 10..100, 
  codepoints = Codepoint.hebrew()
)

val arbList = Arb.list(0..100, Arb.double())
```

---

# Map

```kotlin [map like on lists] {1-2|4|all}
val arbNames = Arb.list(0..100, Arb.string())
val arbUniqueNames = arbNames.map { it.distinct() }

val arbNumbersThatMightBeDivisibleBy3 = Arb.int().map { it * 3 }
```

---

# FlatMap

```kotlin [flatMap]
val arbAge = Arb.int(0..100)
val arbName = Arb.string()

val arbPerson = arbAge.flatMap { 
  age -> arbName.map { 
    name -> Person(name, age)
  }
}
```

---

# Better ways for building complex types

<v-click>

```kotlin [Applicative]
val arbPerson = Arb.bind(
  Arb.int(0..100),
  Arb.string()
) { name, age -> Person(name, age) }
```

</v-click>

<v-click>

```kotlin [Pseudo-Imperative]
val arbPerson = arbitrary {
  val age = Arb.int(0..100).bind()
  val name = Arb.string().bind()
  
  Person(age, name)
}
```

</v-click>

<v-click>

...Arbitraries are [monads](https://www.adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) 😜

</v-click>

---
class: text-2xl
---

# Filtering

```kotlin [filter] {1|3-}
val numbersDivisibleBy3 = Arb.int().filter { it % 3 == 0 }

// you can only filter on one arbitrary!
// therefore we combine two into one to be able to filter some values
val distinctDates = Arb.pair(Arb.localDate(), Arb.localDate())
  .filter { (x, y) -> x != y }
```

<v-click>

- works like on lists
- complex filters hard to read
- can starve on too many discarded samples

</v-click>

---
class: loose-list text-2xl
---

# Favor preconditions


```kotlin [precondition]
checkAll(Arb.localDate(), Arb.localDate()) { first, second ->
  assume(first != second)
  ...
}
```
<v-clicks>

- ✅ supported by most frameworks
- ✅ fail on discard ratio (eg. 10%)!
- ⚠️ inconsistent naming (pre, discard, assume)
- 🤔 use on single arbitrary?

</v-clicks>

---
class: loose-list
---

# Reflection

```kotlin
// let the framework do everything
val arbPerson = Arb.bind<Person>()
```

<v-click>

- ⚠️ limited support between languages & ecosystems
- ⚠️ slow, especially in lists

</v-click>

---

# Unions

<v-click>

```kotlin [mix-in nulls]
val arbMaybeId = Arb.uuid().orNull()
```
</v-click>

<v-click>

```kotlin [combine two arbitraries]
val arbNumbers = Arb.int(1..10).merge(Arb.int(100..150))
```
</v-click>

<v-click>

```kotlin [union types]
// Cat and Dog are both subtypes of Pet
val arbCat = Arb.bind<Cat>()
val arbDog = Arb.bind<Dog>()

val arbPet = arbCat.merge(arbDog)
```

</v-click>

---

# Statistics

are the arbitrary values well distributed for the case?

<v-click>

```kotlin
it("numbers") {
  checkAll(Arb.int()) { number ->
    when {
      number % 2 == 0 -> collect("EVEN")
      else -> collect("ODD")
    }
    
    ... 
  }
}
```

</v-click>

<v-click>

```log
Statistics: [numbers] (1000 iterations, 1 args) 

ODD                                                           502 (50%)
EVEN                                                          498 (50%)
```

</v-click>

<v-click>

...Some frameworks allow testing the distribution 🤔

</v-click>

---
layout: cover
---

# Real life examples

---
class: text-2xl loose-list
---

# Username to Color

- 💡 get a deterministic color for each user
- 💡 for company branding out of a palette
- 💣 nasty modulo operation failure for negative hash values

---
class: text-2xl loose-list
---

# EWMA

- 💡 exponentially weighted moving average
- 💡 more recent values are weighted higher than older ones
- 💡 used to calculate the probable value of a forcasted payment
- 💣 sum to Infinity bug in the function!

---
class: text-2xl loose-list
---

# Generating recurring payments

- 💡 user can create planned payments
- 💡 specify a period, start and optional end
- 💣 off-by-one error when stars align

---
class: text-2xl loose-list
---

# Calculating work days

- 💡 given two dates, how many real work days are there?
- 💡 excluding weekends and German holidays
- 💡 using fixed dates and the [Gauss' Easter Algorithm](https://en.wikipedia.org/wiki/Date_of_Easter#Gauss's_Easter_algorithm)
- 💣 detected the whole thing was too slow to be feasible

---
layout: cover
---

# Real life tips

---
layout: image-right
class: fade text-2xl
image: /tips.png
---

<v-clicks>

- start with example tests
- separate business logic from framework/collaborators
- fine-tune number of iterations
- group your example and property tests
- check if your property tests can actually fail

</v-clicks>

---
layout: cover
---

# Bonus

---
class: text-2xl
---

# Using arbitraries just for random data

```kotlin
val arbCat = Arb...

// not arbitraries, but concrete samples
val randomCat = arbCat.single()
val listOfRandomCats = arbCat.take(100).toList()
```

<v-click>

⚠️ Be careful, **fuzzy tests** are hard to reproduce/understand!

</v-click>

---
layout: cover
---

# Conclusion

---
class: text-2xl fade
---

# Finding properties

<v-clicks>

- business domain rule
- idempotency
- isomorphism
- invariants
- test oracle

</v-clicks>

<v-click>

https://blog.johanneslink.net/2018/07/16/patterns-to-find-properties/

</v-click>

<style>
  li {
    @apply text-3xl line-height-relaxed;
  }
</style>

---
layout: fact
---

... stick to the obvious ones, you will be surprised by the random bugs 😜

<style>
p {
  @apply line-height-relaxed;
}
</style>

---
class: text-3xl fade
---

# When not to use PBT?

<v-clicks>

- slow SUT, eg. framework code
- barely any domain logic
- not enough example tests present
- example tests are good enough
- you don't care about quality 😁

</v-clicks>

---
layout: image-right
image: /pbt-frameworks.png
---

# PBT Frameworks

- Haskell: [Quick Check](https://www.cse.chalmers.se/~rjmh/QuickCheck/manual.html)
- Python: [Hypothesis](https://hypothesis.readthedocs.io/en/latest/)
- JS+TS: [Fast Check](https://fast-check.dev/)
- JVM: [jqwik](https://jqwik.net/)
- Kotlin: [kotest property](https://kotest.io/docs/proptest/property-based-testing.html)
- .NET: [FsCheck](https://fscheck.github.io/FsCheck/)
- C++: [RapidCheck](https://github.com/emil-e/rapidcheck)

... Just search PBT + $LANGUAGE for more!

---
class: text-2xl
---

# Further Resources

- <mdi-youtube/> [Property-Based Testing with jqwik in Java and Kotlin by Johannes Link](https://www.youtube.com/watch?v=dPhZIo27fYE)
- <mdi-youtube/> [Property-Based Testing for everyone by Romeu Moura](https://www.youtube.com/watch?v=5pwv3cuo3Qk)
- <mdi-blog/> [Johannes Link's Blog Posts about PBT (not only in Java)](https://blog.johanneslink.net/tags/#property-based-testing)
- <mdi-blog/> [Comparison between Kotest and Jqwik from 2022 by me](https://www.welcz.de/blog/2022/05/09/jqwik-at-first-glance-from-a-kotest-users-perspective/)
- <mdi-slideshow/> [Property-Based Testing in .NET by Patrick Drechsler](https://draptik.github.io/2025-07-socrates-de-property-based-testing)
- <mdi-github/> [Color Picker in TS using Fast Check and Vitest by me](https://github.com/enolive/color-picker/)
- <mdi-github/> [Mars Rover kata in Haskell using QuickCheck by me](https://github.com/enolive/haskell-mob/blob/solution/mars-rover/test/MarsRoverSpec.hs)

---
src: ./thanks.md
---