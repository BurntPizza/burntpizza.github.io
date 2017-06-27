---
layout: post
title:  "Pairs: Adventures in Nerdsnipery"
date: 2017-06-27
---
<style>
code {
  display: inline-block;
}
span[ib] {
  display: inline-block;
  width: 100px;
}
[serif] {
  font-family: serif;
}
[pseudo-code] {
  font-family: monospace;
}
[indent] {
  text-indent: 20px;
}
[italic] {
  font-style: italic;
}
</style>

  
{::options parse_block_html="true" /}

<div style="font-size: 75%; text-align: center">
(via [xkcd](https://xkcd.com/356/))
</div>

![Chart]({{ site.github.url }}/assets/pairs2.svg)  

## Introduction
So the other day I was reading Cracking the Coding Interview (a great book), and noticed that one of the example programming problems was one that I had done recently on [Hackerrank](http://hackerrank.com). I mentioned this to my friend Ethan sitting next to me, and we got into a discussion.  
  

##  The Problem
First off, the problem goes like this:  
<div serif>
>Given N integers, count the number of pairs of integers whose difference is K.  
>Contraints:  
>N > 1  
>0 < K < 10 <sup>9</sup>  
>The integers are greater than zero and at least K less than u32::MAX.
</div>  

This is a pretty simple problem. Here was my take on it:  
  
```rust
fn count_pairs(numbers: &[u32], k: u32) -> u32 {
    let mut numbers: Vec<u32> = numbers.to_owned();

    numbers.sort();

    let mut pairs = 0;
    let mut i = 0;
    let mut j = 1;

    while j < numbers.len() {
        while j < i || numbers[j] - numbers[i] < k {
            j += 1;

            if j == numbers.len() {
                return pairs;
            }
        }

        if numbers[j] - numbers[i] == k {
            pairs += 1;
        }

        i += 1;
    }

    pairs
}
```  
  
Easy. Just sort the numbers and scan the data, keeping two pointers at where there could be numbers that pair.  
  
This is fast because we don't have to look at the whole data set for each number in it, looking for any pairs, just a small portion.  
  
What else do you need?

## Play It by the Book
Well, both Ethan and the book (and Hackerrank, actually) quickly pointed out that they did it differently: with a hashtable.  
  
Instead of sorting the data to get faster searching, a hashtable can provide membership queries (which, if you think about it, is really what we want) in O(1) time. That's fast!  
  
```rust
fn count_pairs_hash(numbers: &[u32], k: u32) -> u32 {
    let set: HashSet<u32> = numbers.into_iter().cloned().collect();

    let mut count = 0;

    for &x in numbers {
        if set.contains(&(x + k)) {
            count += 1;
        }
    }

    count
}
```  
Nice. This does yield a lower overall time complexity: O(n) instead of O(n log n).  
But, in my pride, I wasn't going to admit that using the hashtable was *better*. Oh no, I *knew* I could do better. If not in time complexity, then in actual runtime.  

## Breaking the Sort Barrier
My first counterpoint: hashtable lookups aren't terribly cache friendly, while on the other hand, a linear scan over contiguous data is just about the fastest, most cache friendly thing a computer can do. And indeed, the sort-and-search method is pretty competitive:  
  
```
$ cargo bench

running 8 tests
test bench_pairs_hash_100            ... bench:       4,418 ns/iter (+/- 84)
test bench_pairs_hash_1000           ... bench:      35,315 ns/iter (+/- 1,723)
test bench_pairs_hash_10_000         ... bench:     418,531 ns/iter (+/- 21,241)
test bench_pairs_hash_100_000        ... bench:   5,934,765 ns/iter (+/- 157,809)

test bench_pairs_sort_search_100     ... bench:       1,604 ns/iter (+/- 93)
test bench_pairs_sort_search_1000    ... bench:      26,254 ns/iter (+/- 1,459)
test bench_pairs_sort_search_10_000  ... bench:     443,333 ns/iter (+/- 23,226)
test bench_pairs_sort_search_100_000 ... bench:   6,204,882 ns/iter (+/- 349,145)
```

We can see that the linear search is very fast on small inputs, but the sorting doen't scale as well as the hashtable, as expected by the higher time complexity.  
  
So now deal with that pesky time complexity, my ace in the hole: Radix sort.  
  
Unlike most sorting algorithms, [Radix sort](https://en.wikipedia.org/wiki/Radix_sort) is a non-comparative sort, meaning it doesn't work by comparing objects in an ordering. This means it can get around the O(n log n) time complexity limit of comparative sorts. Radix sort is instead O(nw), where w is the *width* of the type being sorted. In this case, integers, it is the number of bits. Since the number of bits is a constant, the complexity can be reduced to simply O(n). Ha *ha!*  
  
Fortunately for me, there is already a radix sort library available for Rust, and other than importing it, I only have to change one line:  

```rust
numbers.rdxsort();
```  

Cool. But how does it do?  
  
```
$ cargo bench

running 4 tests
test bench_radix_sort_search_100     ... bench:       2,710 ns/iter (+/- 100)
test bench_radix_sort_search_1000    ... bench:      17,607 ns/iter (+/- 436)
test bench_radix_sort_search_10_000  ... bench:     162,929 ns/iter (+/- 5,834)
test bench_radix_sort_search_100_000 ... bench:   2,435,461 ns/iter (+/- 157,468)
```  
  
Bam! Roughly twice as fast as the hashtable. What now, brown cow?  
  
## ~~Devil's~~ Hashtable's Advocate  
Ok, so confession time: this benchmark might be somewhat disingenuous. (Then again, aren't they always?)  
The Rust standard libray HashSet implmentation uses a hash function called SipHash by default. [SipHash](https://en.wikipedia.org/wiki/SipHash) is a fair bit slower than other hash functions one might consider for use in a hashtable, especially when using integer keys, although for a valid reason.  But let's try something faster:  
  
```rust
let set: fxhash::FxHashSet<u32> = numbers.into_iter().cloned().collect();
```

FxHash is apparently a function derived from one used internally in Firefox, and is used in the Rust compiler itself. It's very fast, and provides better distribution and collision properties than other speedy hashes like FNV. Let's see:  
  
```
$ cargo bench

running 4 tests
test bench_pairs_fxhash_100            ... bench:       1,092 ns/iter (+/- 27)
test bench_pairs_fxhash_1000           ... bench:      10,507 ns/iter (+/- 397)
test bench_pairs_fxhash_10_000         ... bench:     194,419 ns/iter (+/- 5,827)
test bench_pairs_fxhash_100_000        ... bench:   2,238,003 ns/iter (+/- 107,430)
```  

Oh wow, that's crazy. Alright, I guess I have to throw in the towel.  
Right? *Right?*  
  
## Caveats  
  
Ok, so FxHash's result is *also* somewhat disingenuous! The reason SipHash is slow (comparatively) is that it is resistant against collision attacks. Said another way, it is possible, when using FxHash (or most hash functions used in hashtables, actually!) to craft an input that would cause builing the hashtable to blow up to O(n<sup>2</sup>) time! This has been demonstrated to be usable for DDOS attacks on web servers. So *technically* the worst-case time complexity on most hashtable implementations of the Pairs problem is actually quadratic.  
  
Lastly, ~~so I could take my rightful place as pair counting overlord~~ I wanted to see how it went when using a low K value (in this case, 15):  
  
```
$ cargo bench

running 12 tests
test bench_pairs_fxhash_100          ... bench:       1,062 ns/iter (+/- 40)
test bench_pairs_fxhash_1000         ... bench:      10,538 ns/iter (+/- 267)
test bench_pairs_fxhash_10_000       ... bench:     194,001 ns/iter (+/- 4,943)
test bench_pairs_fxhash_100_000      ... bench:   2,224,651 ns/iter (+/- 67,599)

test bench_pairs_sort_search_100     ... bench:       1,639 ns/iter (+/- 29)
test bench_pairs_sort_search_1000    ... bench:      26,371 ns/iter (+/- 1,050)
test bench_pairs_sort_search_10_000  ... bench:     435,932 ns/iter (+/- 11,365)
test bench_pairs_sort_search_100_000 ... bench:   5,524,705 ns/iter (+/- 211,018)

test bench_radix_sort_search_100     ... bench:       2,689 ns/iter (+/- 68)
test bench_radix_sort_search_1000    ... bench:      17,368 ns/iter (+/- 636)
test bench_radix_sort_search_10_000  ... bench:     158,876 ns/iter (+/- 3,629)
test bench_radix_sort_search_100_000 ... bench:   1,750,755 ns/iter (+/- 44,370)
```

Hashtables beware!  
  
----------  
  
<br>
<br>
Bonus log-log graph:  
  
![Chart]({{ site.github.url }}/assets/pairsloglog.svg)  