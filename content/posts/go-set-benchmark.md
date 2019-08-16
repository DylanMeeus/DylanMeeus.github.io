+++
draft = false 
date = "2019-04-15T22:15:11+02:00"
title = "go set benchmark"
+++

Go does not have a `set` datastructure. A common pattern is to mimick a `set` by using a map where
the key is the "set entry" and the value does not matter. 

In a discussion on hackernews recently, we discussed two similar implementations for a `set`,
namely:

* map[type]bool
* map[type]struct{}

They are almost identical, and whilst I really like the first definition it was brought to my
attention that the second might be favourable for performance reasons. Whilst I do like the
readability of using a `map[string]bool`, I do want to write code that is somewhat performant.
Luckily for us, Go provides us with benchmarking tools which we can use to check this.

I wrote two small functions to check the insertion performance of these. 

```

func BenchmarkBoolSet(b *testing.B) {
    m := make(map[int]bool,0)
    for i := 0; i < b.N; i++ {
        m[i] = true
    }
}

func BenchmarkStructSet(b *testing.B) {
    m := make(map[int]struct{},0)
    for i := 0; i < b.N; i++ {
        m[i] = struct{}{}
    }
}

```

We can easily run this benchmark: 

```
$go test -bench=.
goos: linux
goarch: amd64
pkg: github.com/DylanMeeus/GoPlay/benchmark
BenchmarkBoolSet-4           5695645           234 ns/op          40 B/op          0 allocs/op
BenchmarkStructSet-4         6222277           210 ns/op          34 B/op          0 allocs/op
```

From these results we can tell that the "bool-set" uses a bit more memory and takes a bit more time
compared to our "struct-set". 

While looking at this code, something jumped out at me. Why should we keep creating a
new empty struct if we can just re-use the same struct? We don't care about the `value` of our map, only
about the content. 

So we could define our empty struct outside the loop, which results in the following code:

```
var (
    empty = struct{}{}
)

func BenchmarkStructOutside(b *testing.B) {
    m := make(map[int]struct{},0)
    for i := 0; i < b.N; i++ {
        m[i] = empty 
    }
}
```

When we add this, we get a slight performance inprovement over the previous one. Memory allocation
stayed the same though.

```
BenchmarkBoolSet-4           5695645           234 ns/op          40 B/op          0 allocs/op
BenchmarkStructSet-4         6222277           210 ns/op          34 B/op          0 allocs/op
BenchmarkStructOutside-4     6246123           206 ns/op          34 B/op          0 allocs/op
```

Considering the small difference between these last two methods, I'd say it's probably neglible. 
The difference between the "bool-set" and "struct-set" is slightly larger, so that's something you
might want to keep in mind if you're writing a set somewhere in the hot path of your code. :)
