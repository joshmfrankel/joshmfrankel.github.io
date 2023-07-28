---
layout: post
title: "When is an Array an Array? Strategies for checking Array equality in Ruby"
categories:
- articles
tags:
  - ruby
---

Object equality is an interesting topic. You can check for matching values (with or without ordering). You can
also check to see if the object has the same in-memory id. I've written code to diff two arrays before but
I've never sat down to think about all the ways to accomplish this. Also I was able to explore what I believe
are the most optimal approaches.

<!--excerpt-->

## Basic equality
The most basic example of checking two objects is the equality method `==`. This is a common check done between Ruby
objects and Array is no different. So what exactly does it do? According to the documentation, it will check the `.size`
of both arrays along with each individual index value against the other array. Looking at the [source code for == in Ruby 3.2.2](https://ruby-doc.org/3.2.2/Array.html#method-i-3D-3D)
we can see this at work:

``` ruby
rb_ary_equal(VALUE ary1, VALUE ary2)
{
    if (ary1 == ary2) return Qtrue;
    if (!RB_TYPE_P(ary2, T_ARRAY)) {
        if (!rb_respond_to(ary2, idTo_ary)) {
            return Qfalse;
        }
        return rb_equal(ary2, ary1);
    }
    if (RARRAY_LEN(ary1) != RARRAY_LEN(ary2)) return Qfalse;
    if (RARRAY_CONST_PTR_TRANSIENT(ary1) == RARRAY_CONST_PTR_TRANSIENT(ary2)) return Qtrue;
    return rb_exec_recursive_paired(recursive_equal, ary1, ary2, ary2);
}
```

The first conditional checks to see if the arrays are identical in memory:

`if (ary1 == ary2) return Qtrue;`

The third conditional is an early return false if the size of the two array's doesn't match:

`if (RARRAY_LEN(ary1) != RARRAY_LEN(ary2)) return Qfalse;`

The last line is a recursive call which I assume is the iteratively check every index's value
against the other array. The remainder of the method (conditional 2 and 4) I'm less sure about but looking
at the code they both can perform early returns to break out of the recursive method call. Assumedly either
when there are no more indexes to check against or the arrays are found not to match.

Now working our way back to the higher-level Ruby implementation, what does a practical example of this look like?

``` ruby
array_1 = [1, 2, 3]

array_1 == [1, 2, 3] #=> true
```

The above checks the two arrays against each other and finds them identical.

There are two other strategies you can take here. Both the `Array#eql?` and `Array#<=>` methods will allow you to check array equality. They work
  slightly differently from each other:

* `Array#eql?` - Similiar to Array#== but utilizes Object#== internally
* `Array#<=>` - Generally used within sort_by blocks to check difference in arrays. Returns 0 for matching, 1 when there are extra indexes, and -1 when there are missing indexes.

Here's a quick example of each:

``` ruby
array_1.eql?([1, 2, 3]) #=> true

# Spaceship operator returns: 0 (equal), -1 (elements less than other array), 1 (elements more than other array)
(array_1 <=> [1, 2, 3]) == 0 #=> true
```

Personally, I find `==` to be the easiest to read. This is all well and good but what happens if the arrays are in different orders?

``` ruby
array_1 = [1, 2, 3]
array_2 = [2, 3, 1]

array_1 == array_2 #=> false
```

Now they are no longer equivalent according to `==`. So from this we can infer that `==` is order-dependent. We'll need
another strategy in order, ;-), to test for equality properly.

## Equality without order dependence

Using our above example, let's say we want the two arrays to return true when checking equality. We don't care if they
are in the same order just that they contain the same elements. We can achieve that simply by first sorting the arrays.

``` ruby
array_1 = [1, 2, 3]
array_2 = [2, 3, 1]

array_1.sort == array_2.sort #=> true
```

The above ensures that the array elements are sorted using the same algorithm keeping them in identical ordering.

Now there is the additional processing overhead of having to first sort the arrays (1 iteration per array) and then check the index
values against the other array (another iteration). This means that it will be slightly less efficient in checking since it
needs to iterate once on each array and then another time to check for equality.

Before we improve efficiency, let's throw another challenge in the mix. What happens if one of the array's contains duplicate elements?

``` ruby
array_1 = [1, 2, 3]
array_2 = [2, 2, 2, 2, 3, 1]

array_1.sort == array_2.sort #=> false
```

Even though array_1 and array_2 have the exact same values within them, array_2 contains duplicates of the value 2. This
fails the equality check.

## Equality without duplicates

Taking a line from the above approach we can add yet another iteration to our arrays ontop of the existing `.sort` to
remove the duplicate values. We achieve this by using `.uniq` which iterates through the array and removes duplicates.

``` ruby
array_1 = [1, 2, 3]
array_2 = [2, 2, 2, 2, 3, 1]

array_1.uniq.sort == array_2.uniq.sort #=> true
```

We've now succeeded in checking the equality between two arrays without regard for order or duplicate values. Internally sort and uniq work like so:

* `#uniq` - "self is traversed in order, and the [first](https://apidock.com/ruby/Array/first) occurrence is kept."
* `#sort` - "Comparisons for the [sort](https://apidock.com/ruby/Array/sort) will be done using the <=> operator"

Now each array undergoes two iterations (sort and uniq) before finally having an equality iteration. This becomes even
less efficient and requires a bit more overhead to understand. There's one more Array strategy I want to discuss next regarding
array intersections before moving onto my preferred strategy.

## Equality with Array intersection

This approach utilizes a lesser used syntax to perform an [array intersection](https://ruby-doc.org/3.2.2/Array.html#method-i-26). An array intersection means to take
two arrays and find the common elements without duplicates. It utilizes the `Array#&` syntax like I've demostrated below:

``` ruby
array_1 = [1, 2, 3]
array_2 = [2, 2, 2, 2, 3, 1]

(array_1 & array_2) #=> [1, 2, 3]
(array_1 & array_2) == array_1 #=> true
```

I've split the above into two steps. The first is performing the intersection on the two arrays. The result of doing this
is a new array that contains common values using the ordering of the first array utilizes. By using that last statement,
we know that if the ordering of the first array is used and duplicates are removed then if the result is equal to the
first array the arrays must be equivalent. Shown as `(array_1 & array_2) == array_1`.

Internally this is iterating through values along with checking for duplicates with `eql?`.

Outside of the Array class there is an alternative which I find to be highly readable in code. Let's take a look at an
improved readability approach.

## Array Equality with Sets

{% include blockquote.html quote="Set implements a collection of unordered values with no duplicates. This is a hybrid of Array's intuitive inter-operation facilities and Hash's fast lookup." source_link="https://ruby-doc.org/stdlib-2.7.1/libdoc/set/rdoc/Set.html" source_text="https://ruby-doc.org/" %}

A Set is like an array but by definition is an enumerable without duplicates and order independent. Therefore we can convert
our arrays to Sets and achieve the same effect as we did with the chained `.uniq.sort` above.

``` ruby
array_1 = [1, 2, 3]
array_2 = [2, 2, 2, 2, 3, 1]

Set.new(array_1) == Set.new(array_2) #=> true
array_1.to_set == array_2.to_set #=> true
```

Very clean looking and readable. Interestingly enough, `Set#==` internally uses `Object#==` identically to have `Array#eql?` achieves
equality checks. [Here's the documentation for `Set#==`](https://ruby-doc.org/stdlib-2.7.1/libdoc/set/rdoc/Set.html#method-i-3D-3D).

## So which strategy is the best?

Honestly, it depends. Personally I believe that the `Set.new` approach above is the most elegant and simplest solution to understand, though there
are merits to the explict straightforwardness of `.uniq.sort` or the slickness of array intersection. Ultimately, its up to you to decide which is best for
your own coding style. I will say that one of the above strategies may prove to be more performant than another. My guess would be
that `Set.new` is the fastest strategy but I have not performed benchmarks to verify this.

Do you have another Array equality strategy I missed? Know of a more efficient method? I'd love to discuss in the comments below.
