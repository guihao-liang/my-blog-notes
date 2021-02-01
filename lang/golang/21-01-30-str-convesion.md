---
title: "Golang: string reading"
subtitle: "Golang memory allocation during conversion between string and slice"
author: Guihao Liang
published: true
bigimg: "/img/golang.jpg"
tags: ['golang']
date: 2021-01-31 17:51:46
---

TL;DR,

1. memory allocation happens immediately when converting between string and slice.
   1. rune slice is meant to random read or write UTF encoded string.
   2. byte slice is meant to write.
   3. no COW (copy-on-write) optimization for reading string through slice.
2. use original ASCII string instead of byte slice for reading.
3. use `range` to sequential read UTF encoded string, instead of rune slice.

## random read

### string to rune slice

To random access [character][char go] in UTF-8 encoded string, it's common to convert str to `rune` slices.

```go
str := "世界"
letters := []rune(str)
```

When we do `letters[1]`, does it decodes UTF character on the fly?

The answer is no. The UTF is **variant** length encoding, so there's no possible way to decode a random position without decoding all positions ahead. Rune slice is not index-wise mapped to underlying bytes of string. That's to say, during construction, all bytes from string will be read and decoded. The construction operation is quite expensive.

What's more, COW (copy-on-write) for rune slice is unnecessary. If you construct a rune slice from string and never read/write it, the optimizer can ignore that construction.

From assembly code like [this](https://godbolt.org/z/8bf69o), golang calls `runtime.stringtoslicerune`, which hides the detail of the memory allocation. Therefore, I turn to golang benchmark tool.

The benchmark setting is below:

```golang
import "testing"

var str string

func init() {
    // 1024B
    tmp := make([]byte, 1024)
    str = string(tmp)
}
```

The template string is 1024 ASCII bytes, which is 1024B or 1MiB in total. As we will see soon, it's easy to figure out how much memory is allocated for each benchmark test.

```golang
func BenchmarkStrToBytes(b *testing.B) {
    ll := len(str)
    for ii := 0; ii < b.N; ii++ {
        newBytes := []byte(str)
        newBytes[ll-1] = 10
    }
}

func BenchmarkStrToRunes(b *testing.B) {
    ll := len(str)
    for ii := 0; ii < b.N; ii++ {
        newRunes := []rune(str)
        newRunes[ll-1] = 10
    }
}
```

In the above 2 benchmark examples, I do extra assignments to make sure memory is allocated if there's any COW. Let's run a benchmark test with `go test -benchmem -bench=.`:

```bash
BenchmarkStrToBytes-12           8774433               131 ns/op            1024 B/op          1 allocs/op
BenchmarkStrToRunes-12            624459              1658 ns/op            4096 B/op          1 allocs/op
```

As we know, `rune` is an alias to `int`, which is quad-word sized (4 bytes). As a result, the rune slice is 4096B, which is indeed **4x** of the corresponding ASCII byte slice.

Rune slice is **only** meant for **random read** or **write** UTF strings. If you use it for a sequential read, it would be memory inefficient, as it's shown above. To sequential read a UTF encoded string, check [here](#sequential-access).

### string to byte slice

Converting string to byte slice for reading is unnecessary. Index operator on the original string is sufficient for random read and `range` for a sequential read.

What I want to discuss here is whether there's COW for byte slice. Unlike rune slice, elements from byte slice are index-wise mapped from string elements. COW is possible.

Let's see if there's any COW provided by golang by just reading the byte slice without any writes.

```golang
func BenchmarkStrToBytesCow(b *testing.B) {
    tmp := func(str string) []byte {
        newBytes := []byte(str)
        return newBytes
    }
    for ii := 0; ii < b.N; ii++ {
        tmp(str)
    }
}
```

Benchmark test result shows allocation happens even there's no write to the byte slice constructed from a string.

```bash
BenchmarkStrToBytesCow-12     9071371     131 ns/op      1024 B/op     1 allocs/op
```

That is to say, **whenever** you initialize byte slice from string, you mean to **write** even if you don't and new memory is allocated.

### string from slice

Whenever you convert slice to string, new memory is allocated immediately. The reasoning is simple: string is immutable, and slice content can be mutated. Therefore, string needs to have a **copy** of the content in case the underlying memory pointed by the byte slice is modified afterward.

```golang
# simple benchmark
func BenchmarkBytesToStr(b *testing.B) {
    tmp := func(bytes []byte) byte {
        newStr := string(bytes)
        return newStr[len(newStr)-1]
    }
    newBytes := []byte(str)
    for ii := 0; ii < b.N; ii++ {
        tmp(newBytes)
    }
}
```

The output for this benchmark test:

```bash
BenchmarkBytesToStr-12     9468996    125 ns/op    1024 B/op     1 allocs/op
```

As it's shown above, there's one allocation for each construction from byte slice to string. No COW is observed since it's troublesome to monitor the memory pointed by the byte slice.

## Sequential access

You can also construct byte or rune slice to iterate contents in the string. That would be **inefficient** since extra memory allocation will drag down the performance.

The better ways are to

1. use original string to read bytes by either `range` or index operator.

2. use `range` for UTF string, which will decode `rune` on the fly for rune slice by calling [runtime.decoderune][range rune].

```golang
func BenchmarkStrToRuneRange(b *testing.B) {
    tmp := func(str string) rune {
        var tmp rune
        for _, ch := range str {
            tmp = ch
        }
        return tmp
    }
    for ii := 0; ii < b.N; ii++ {
        tmp(str)
    }
}
```

Benchmark output:

```bash
BenchmarkStrToRunes-12       624459        1658 ns/op        4096 B/op       1 allocs/op
BenchmarkStrToRuneRange-12  1660866         723 ns/op           0 B/op       0 allocs/op
```

No extra allocation is needed and it's twice faster than the method from the previous section that converts string to rune slice.


[range rune]: https://godbolt.org/z/W95ofb
[char go]: https://blog.golang.org/strings