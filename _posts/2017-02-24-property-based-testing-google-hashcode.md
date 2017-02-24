---
published: true
tags: scala test
title: Property Based Testing when preparing for Google Hashcode
---
## Property Based Testing when preparing for Google Hashcode

### Problem Overview

This week was the qualification round for [Google Hashcode 2017](https://hashcode.withgoogle.com/).

In order to prepare a bit, I started working on the practice problem : [Slice the Pizza]({{site.baseurl}}/assets/pizza.pdf).

It involves slicing a pizza in slices small enough, with enough Tomato and Mushrooms in each slice, and getting as much of the pizza in the slices.

![pizza-hashcode.png]({{site.baseurl}}/assets/pizza-hashcode.png)

Once I have selected some slices, I need to check if other possible slices intersect with the selected ones.
So I had a code similar to this one, which checks if there is a common cell :

```scala
case class Slice(row1: Int, row2: Int, col1: Int, col2: Int) {
  lazy val cells = (for {
    r <- row1 until row2
    c <- col1 until col2
  } yield Point(r, c)).toSet

  def intersects(that: Slice) = cells.exists(that.cells.contains)
}
```

I was concerned about the performance of this code and I wanted to rewrite it using only the `col` and `row` indices.

### Property Based Testing

I wanted to be sure to avoid regressions and that the new implementation of `intersects` returns the same result as the old one in all cases.

This can be achieved easily with property based testing and the [ScalaCheck](https://www.scalacheck.org/) framework ([integrated with ScalaTest](http://www.scalatest.org/user_guide/writing_scalacheck_style_properties)).

Here is the basic test I wanted to write :

```scala
"2 Slices" should "intersect properly" in {
  forAll { (slice1: Slice, slice2: Slice) =>
    slice1.intersects(slice2) shouldBe slice1.cells.exists(slice2.cells.contains)
  }
}
```

### Generating test instances of `Slice`

By adding [scalacheck-shapeless](https://github.com/alexarchambault/scalacheck-shapeless) as described in this [sample project](https://github.com/tyrcho/scalatest-scalacheck-demo), we can have ScalaCheck generate automatically instances of case classes. 
```xml
<dependency>
  <groupId>com.github.alexarchambault</groupId>
  <artifactId>scalacheck-shapeless_1.13_2.11</artifactId>
  <version>1.1.1</version>
  <scope>test</scope>
</dependency>
```

But this is not very usefull here since a `Slice` only makes sense for specific rows and cols.

So it's better to control the generation of our own `Slice`s, with rows and cols between 1 and 200, and be sure that `row2 > row1`.

```scala
implicit val arbitrarySlice = Arbitrary {
  for {
    row1 <- Gen.chooseNum(1, 200)
    row2 <- Gen.chooseNum(row1 + 1, 200)
    col1 <- Gen.chooseNum(1, 200)
    col2 <- Gen.chooseNum(col1 + 1, 200)
  } yield Slice(row1, row2, col1, col2)
}
```

### New implementation of `intersects`

With this in place, I can change the implementation of `intersects`. I made several tries before arriving to this one, hence the utility of my test !

```scala
 def intersects(that: Slice) = {
    @inline def between(a: Int, b: Int, x: Int) = x >= a && x <= b
    @inline def intersects1d(x1: Int, x2: Int, a1: Int, a2: Int) =
      between(x1, x2, a1) || between(x1, x2, a2) ||
        between(a1, a2, x1) || between(a1, a2, x2)

    intersects1d(col1, col2 - 1, that.col1, that.col2 - 1) && intersects1d(row1, row2 - 1, that.row1, that.row2 - 1)
  }
```

I could have checked that the new implementation is really faster with a [ScalaMeter](https://scalameter.github.io/) microbenchmark but this is another story.

### Shrinking

ScalaCheck can perform "shrinking" on the instances generated which fail your test. 
This gives you better error messages with simpler instances.

It does it automatically for basic types (`List`s, `Int` ...) but you need to provide a `Shrink`er for your own classes. It's rather simple when you reuse the existing `shrink` method which takes a Tuple.

```scala
implicit val shrinkSlice: Shrink[Slice] = Shrink {
  case Slice(a, b, c, d) => for {
    (a1, b1, c1, d1) <- shrink((a, b, c, d))
  } yield Slice(a1, b1, c1, d1)
}
```

You can find the [full test file](https://github.com/wl-seclin-hashcode/hashcode-2017-practice/blob/master/src/test/scala/hashcode/training/SliceSpec.scala) on my [Github project](https://github.com/wl-seclin-hashcode/hashcode-2017-practice). 
