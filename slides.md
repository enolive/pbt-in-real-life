---
# You can also start simply with 'default'
theme: enolive
# some information about your slides (markdown enabled)
title: Property-Based Testing IRL
info: |
  ## Property-Based Testing IRL
  ...not only for mathematicians!
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph
seoMeta:
  # By default, Slidev will use ./og-image.png if it exists,
  # or generate one from the first slide if not found.
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
- duplication solvable with parameterizing the test

</v-clicks>

<style>
.slidev-layout {
  @apply gap-x-4xl;
}
</style>

---
layout: two-cols-header
class: fade
---

# Property-Based Testing

::left::

```kotlin 
[property]
it("numbers divisible by 3 produce a Fizz") {
  checkAll(Arb.int().filter { it % 3 == 0 }) { input ->
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

<style>
.slidev-layout {
  @apply gap-x-4xl;
}
</style>

---
layout: image-right
image: /research.png
class: fade
---

# Some other properties

<v-clicks>

- numbers divisible by 5 produce a Fizz
- result is never empty
- pattern repeats every 15 numbers
- function works for any possible number

</v-clicks>

---
layout: image-right
image: /stop.png
class: fade
---

# What is Property testing **not**?

<v-clicks>

- replacement for Example Testing
- exhaustive testing
- prove your code is correct
- only for mathematicians
- only for complicated algorithms

</v-clicks>

---
layout: fact
class: fade
---

<v-clicks>

- Tests are **hypotheses** about the correctness of the code.
- Therefore, they **cannot be proven**, only **falsified**.
- Finding a good, **falsifiable** hypothesis is the hard thing!

</v-clicks>

---
layout: cover
---

# How it works

---
class: text-4xl fade
---

<v-clicks>

- each property test is executed many times
- on failure
- try to find a simpler example
- display used seed to deterministically reproduce the error

</v-clicks>

---

# Can you guess the error?

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

```kotlin
val constant = Arb.constant(42)

val arbNumber = Arb.int()

val arbStatus = Arb.enum<Status>()
// to my knowledge, kotest is the only PBT framework having exhaustive generators
val exStatus = Exhaustive.enum<Status>()

val arbNumberPositive = Arb.int(min = 0)

val arbString = Arb.string()

val arbHebrewString = Arb.string(
  range = 10..100, 
  codepoints = Codepoint.hebrew()
)

val arbList = Arb.list(0..100, Arb.double())
```

---

# Map & FlatMap

<v-click>

map works like on lists

```kotlin
val arbNames = Arb.list(0..100, Arb.string())
val arbUniqueNames = arbNames.map { it.distinct() }
```

</v-click>

<v-click>

...flatMap as well

```kotlin
val arbAge = Arb.int(0..100)
val arbName = Arb.string()

val person = arbAge.flatMap { 
  age -> arbName.map { 
    name -> Person(name, age)
  }
}
```

</v-click>

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

...Arbitraries are monads 😜

</v-click>

---
class: text-2xl fade
---

# Filtering

<v-clicks>

- filter like on lists
- filter harder to read
- can starve on too many discarded samples

</v-clicks>

<v-click>

```kotlin [filter]
// you can only filter on one arbitrary!
// therefore we combine two into one to be able to filter some values
val distinctDates = Arb.pair(Arb.localDate(), Arb.localDate())
  .filter { (x, y) -> x != y }
checkAll(distinctDates) { (first, second) ->
  ...
}
```
</v-click>

---
class: fade loose-list
---

# Better use assumptions


```kotlin [assumption]
checkAll(Arb.localDate(), Arb.localDate()) { first, second ->
  assume(first != second)
  ...
}
```
<v-clicks>

- ⚠️ fail on discard percentage (default usually 10..20%)!
- ⚠️ not all frameworks support assumptions 😭

</v-clicks>

---
class: loose-list fade
---

# Reflection

```kotlin
// let the framework do everything
val arbPerson = Arb.bind<Person>()
```

<v-clicks>

- ⚠️ limited support between languages & ecosystems
- ⚠️ slow, especially in lists

</v-clicks>

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

```
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
class: text-2xl loose-list fade
---

# Username to Color

<v-clicks>

- 💡 get a deterministic color for each user
- 💡 for company branding out of a palette
- 💣 nasty modulo operation failure for negative hash values

</v-clicks>

---
class: text-2xl loose-list fade
---

# EWMA

<v-clicks>

- 💡 exponentially weighted moving average
- 💡 more recent values are weighted higher than older ones
- 💡 used to calculate the probable value of a forcasted payment
- 💣 sum to Infinity bug in the function!

</v-clicks>

---
class: text-2xl loose-list fade
---

# Generating recurring payments

<v-clicks>

- 💡 user can create planned payments
- 💡 specify a period, start and optional end
- 💣 off-by-one error when stars align

</v-clicks>

---
class: text-2xl loose-list fade
---

# Calculating work days

<v-clicks>

- 💡 given two dates, how many real work days are there?
- 💡 excluding weekends and German holidays
- 💡 using fixed dates and the [Gauss' Easter Algorithm](https://en.wikipedia.org/wiki/Date_of_Easter#Gauss's_Easter_algorithm)
- 💣 detected the whole thing was too slow to be feasible

</v-clicks>

---
layout: cover
---

# Real life tips

---
class: fade text-2xl
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
class: fade
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

# PBT Frameworks

- Haskell: [Quick Check](https://www.cse.chalmers.se/~rjmh/QuickCheck/manual.html)
- Python: [Hypothesis](https://hypothesis.readthedocs.io/en/latest/)
- JS+TS: [Fast Check](https://fast-check.dev/)
- JVM: [jqwik](https://jqwik.net/)
- Kotlin: [kotest property](https://kotest.io/docs/proptest/property-based-testing.html)
- .NET: [FsCheck](https://fscheck.github.io/FsCheck/)

... Just search PBT + $LANGUAGE for more!

---
class: text-2xl
---

# Further Resources

- <mdi-youtube/> [Property-Based Testing with jqwik in Java and Kotlin by Johannes Link](https://www.youtube.com/watch?v=dPhZIo27fYE)
- <mdi-youtube/> [Property-Based Testing for everyone by Romeu Moura](https://www.youtube.com/watch?v=5pwv3cuo3Qk)
- <mdi-blog/> [Johannes Link's Blog Posts about PBT (not only in Java)](https://blog.johanneslink.net/tags/#property-based-testing)
- <mdi-blog/> [comparison between kotest and jqwik from 2022 by me](https://www.welcz.de/blog/2022/05/09/jqwik-at-first-glance-from-a-kotest-users-perspective/)
- <mdi-slideshow/> [Property-Based Testing in .NET by Patrick Drechsler](https://draptik.github.io/2025-07-socrates-de-property-based-testing)
- <mdi-github/> [color picker in TS using fastcheck by me](https://github.com/enolive/color-picker/)

---
src: ./thanks.md
---