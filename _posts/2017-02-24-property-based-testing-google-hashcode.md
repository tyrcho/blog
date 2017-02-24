---
published: true
---
## Property Based Testing when preparing for Google Hashcode

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

I was concerned about the performance of this code and I wanted to rewrite it using only the col and row indices.

