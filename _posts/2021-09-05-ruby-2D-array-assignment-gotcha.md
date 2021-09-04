---
layout: posts
title:  "Ruby 2D array assignment gotcha"
date:   2021-09-04 16:57:47 +0530
layout: single
author_profile: true
comments: true
tags: [ruby, quirks, array]
---
Recently I came across an interesting ruby Array class gotcha.

As usual, I will start with the example.

Let us instantiate a 2D array.

``` ruby
arr = Array.new(3, Array.new(3, 0))
```

Now, we can assign a value to a cell in the matrix

``` ruby
arr[1][1] = 1
puts arr
# [[0, 1, 0], [0, 1, 0], [0, 1, 0]]
```

Huh! (scratches head)

Let us try without using `Array.new`

``` ruby
arr_2 = []
3.times { arr_2 << Array.new(3, 0)}
arr_2[1][1] = 1
puts arr_2
# [[0, 0, 0], [0, 1, 0], [0, 0, 0]]
```

This looks ok.

Time to hit the [docs](https://docs.ruby-lang.org/en/3.0.0/Array.html)

> Note that the second argument populates the array with references to the same object. Therefore, it is only recommended in cases when you need to instantiate arrays with natively immutable objects such as Symbols, numbers, true or false.

To declutter our initial `arr` instantiation, the row was initialized only once and used multiple times.

``` ruby
row = Array.new(3, 0)
arr = Array.new(3, row)
```

We can check some more

``` ruby
arr = Array.new(3, Array.new(3, 0))
arr[0].equal? arr[1]   # will return true
arr[0].equal? arr[2]   # will return true

arr_2 = []
3.times { arr_2 << Array.new(3, 0)}
arr_2[0].equal? arr_2[1]   # will return false
arr_2[0].equal? arr_2[2]   # will return false
```

There is a safer alternative available. **We can use blocks**

``` ruby
arr = Array.new(3) { Array.new(3, 0) }
arr[0].equal? arr[1]   # will return false
arr[0].equal? arr[2]   # will return false

arr[1][1] = 1
puts arr
# [[0, 0, 0], [0, 1, 0], [0, 0, 0]]
```
