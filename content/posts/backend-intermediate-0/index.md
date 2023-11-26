---
author: "Abhay"
title: Bloom filters, the magic of O(1) indexes with rust
description: "Bloom filters are probabilistic data-structures, what that means it won’t be right all of the time but most of the time. Why would we use such a thing, cause it BLAZINGLY fast!! We will explore this with Rust cause it’s great and not as hard as you might think."
date: "2023-11-26"
template: "post"
draft: false
slug: "backend-intermediate-0"
series:
  - "backend-intermediate"
tags:
  - "backend"
  - "rust"
  - "low-level-design"
hidemeta: false
comments: false
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "posts/backend-intermediate-0/thumbnail-bi0.png" # image path/url
    alt: "Thumbnail: Probably yes, definitely no. - Bloom filters" # alt text
    caption: "Bloom filters in a sentence." # display caption under cover
    relative: true # when using page bundles set this to true
    hidden: false # only hide on current single page
---
## Bloom filters
Bloom filters are probabilistic data-structures, what that means it won’t be right all of the time but most of the time. Why would we use such a thing, cause it **BLAZINGLY** fast!!

We will explore one simple and one Production-ish implementation of this with Rust cause it’s a great language and not as hard as you might think.

I won't dive into the history and mathematics of bloom filters, [this wikipedia page](https://en.wikipedia.org/wiki/Bloom_filter) does that great. I will be focusing more on intuition of how they work and how to make one. You can find the code at the [backend-rs github repo](https://github.com/rathod-sahaab/backend-rs/tree/main/data-structures/bloom-filter/src)

So without further adieu.

## Probably yes, definitely no.
Bloom filter is used to "store" whether it has seen something already, similar to some applications where you would use `HashSet`, `unorderd_map`.
When you ask it whether it has seen something it will respond with a yes or a no (boolean true/false).

No is straight forward, it has never seen the queried data.
But, When it says yes, there's a chance for false positive. That means bloom filter might say it has seen some data, which it in fact never saw.

Why would we use a bloom filters because it's fast  $ O(1) $ for already hashed data. So when bloom filters gives go ahead you can run expensive lookups like DB indexes $ O(log N) $ and others which give exact result. And in a situation when there are many **No** cases, that saves a lot of resources.

### The contract
In rust we represent contracts/interfaces with traits. Here we define a trait `BloomFilter` anything claiming to be a `BloomFilter` must implement these functions.
```rust
pub trait BloomFilter {
    fn insert(&mut self, key: &str);

    /// probably yes, definitely no.
    fn contains(&self, key: &str) -> bool;
}
```
Here the key is a text we want to insert and later see if it was seen.

<details>
  <summary>Explain Rust</summary>

<br/>

Insert modifies the data in the data structure, flips bits etc. So it must modify some data in the data structure.
```rust
fn insert(&mut self, key: &str);
          // ^             ^--- &str: string reference*
          // +--- &: reference, mut: modifies, self: this equivalent
```

Contains should not modify data-structure, or else calling multiple times with same params will return different result
```rust
fn contains(&self, key: &str) -> bool;
          // ^             ^      ^--- true: probably yes, false: definitely no.
          // |             |--- &str: string reference*
          // +--- &: reference, self: this equivalent
          //         omitted mut as contains does not modify self
```
</details>

## Implementation
Bloom filter are generally implemented as an array/vector of bits. I will use array of 32 booleans as an example.

```rust
pub struct BloomFilter32 {
    bits: [bool; 32],
}
```

### hash function
A hash function is a crucial component of most bloom filters, what a hash function does is it takes a lot of data and returns wildly smaller input data dependent output. A lot of information is lost in the process but, preserving data was never the aim, aim was to reduce the data so it is easy to process.

A good hash function for bloom filters would have properties like:
1. Even distribution.
2. String sensitive. (order, case, etc. sensitive)
3. Same result for same input.


In our case hash function will take a string, and return a number between 0-31 (an index from our bits array).
A simple hash function would be one that add all characters and mods them by 32. Let's implement that.
```rust
// v--- impl block, way to define methods/member function in rust
impl BloomFilter32 {
//                     v---  missing self, static function
    fn additive_hasher(key: &str, seed: usize) -> usize {
        // fold:
        //    Produce one value from all characters
        key.chars().fold(0, |acc, ch| -> usize {
            (acc + seed + (ch as usize % 32)) % 32 // modulo maths result 0-31
        })
    }
}
```

`additive_hasher` also takes a seed to be able to return different hash for same data, thanks to modulo mathematics. Which gives us a lot of different hash functions for next to no code!!

### impl BloomFilter
Insert:
1. When we insert string, we ask our hash function for an index (data-dependent, evenly distributed, seemingly random).
2. We mark our bit at that index as true/1.

```rust
impl BloomFilter for BloomFilter32 {
    fn insert(&mut self, key: &str) {
        let hash = Self::additive_hasher(key, 0);

        self.bits[hash_a] = true;
    }
```


Contains:
1. We are passed the string, so we again ask the hash function for an index (will be same for same data always).
2. If the bit at given index is true that means same hash value was encountered before, we return true(probably yes).
3. If the bit at given index is false that means same hash value was never encountered before ,we return false(definitely no).

```rust
fn contains(&self, key: &str) -> bool {
    let hash = Self::additive_hasher(key, 0);

    self.bits[hash] // <-- no semicolon means return in rust
}
```



You might be wondering since the start why probably yes, definitely no.
Because two different strings can have same hash value(index). So if an index is true, it means queried string or some other string which had the same value was seen.

Example:
Let's say our hash function counts number of `a`s in a string. Hash of `hash(apple)=1`, `hash(banana) = 3`, `hash(mango)=1`. So if bit at index `1` is true, it can be due to either `apple` or `mango`, so we can't say for sure that `apple` was seen, `apple` was probably seen:

To minimise this false positives we use multiple hash functions which will give us different indexes.

In our example:
`hash_a_count(apple), hash_first_last_sum(apple) = 1, 6 (1 + 5)`
`hash_a_count(mango), hash_first_last_sum(mango) = 1, 28 (13 + 15)`
`hash_a_count(banana), hash_sum(banana) = 1, 3 (2 + 1)`

As you can see the chance of collision is lowered. Adding that in code (remember our hasher takes a seed!).

```rust
impl BloomFilter for BloomFilter32 {
    fn insert(&mut self, key: &str) {
        let hash_a = Self::additive_hasher(key, 0);
        let hash_b = Self::additive_hasher(key, 1);

        self.bits[hash_a] = true;
        self.bits[hash_b] = true;
    }

    fn contains(&self, key: &str) -> bool {
        let hash_a = Self::additive_hasher(key, 0);
        let hash_b = Self::additive_hasher(key, 1);

        self.bits[hash_a] && self.bits[hash_b]
    }
}
```

And that's it our simple bloom filter is implemented.

#### Simple bloom filter run test
```rust
// ...imports

fn main() {
    let mut bl = BloomFilter32::default();

    bloomy(&mut bl);
}

fn bloomy(bl: &mut impl BloomFilter) {
    let keys = vec!["mango", "apple", "orange", "banana"];

    // insert elements in bloom filter (using lambda function)
    keys.iter().for_each(|&key| bl.insert(key));
    //         take(ref) ---^              ^use

    let mut test_keys = vec!["carrot", "radish", "vegetable", "onion"];
    test_keys.extend(keys);

    let results = test_keys.iter().map(|&key| bl.contains(key));

    // ..code for ouput
}
```
Result
```
+-----------+-------+
| carrot    | false |
+-----------+-------+
| radish    | false |
+-----------+-------+
| vegetable | false |
+-----------+-------+
| onion     | false |
+-----------+-------+
| mango     | true  |
+-----------+-------+
| apple     | true  |
+-----------+-------+
| orange    | true  |
+-----------+-------+
| banana    | true  |
+-----------+-------+
```
You might have noticed our function `bloomy` doesn't depend on `BloomFilter32`, and this is where interfaces shine and so does rust because rust does types best. We can replace BloomFilter32 with anything so let's replace it with something good.

## Production-ish implementation
Now don't go using `BloomFilter32` in your production environment, there is more to bloom filters.
### Data bits
We used boolean to store our data bits, which wastes space as a boolean value takes 1 byte in memory. To fix this we will use `bitvec` crate for rust, which in effect stores 8 bits of accessible information per byte. We just tell `bitvec` how many bits we want and it takes care of the rest.
```rust
use bitvec::prelude::*;

pub struct BloomFilterProd {
    bits: BitVec,
    hash_count: usize,
}
```
But how many bits do we want?

#### Bit count

We used 32 bits of data for our bloom filter, now as we keep adding more elements, more and more of the bits will "on". At one point all our bits will be filled and every query will return probably yes i.e. Rate of error will increase. This indicates us that the number of data bits depends on the number of elements we are planning to "store".
Think of it this way, the most amount of elements if every thing goes correctly is 32, each bit for presence of a distinct element.

There is a great proof on wikipedia page for the exact mathematics, the formula we will be using is
$$
m = -n\frac{\log_2\epsilon}{\log2}
$$

- **m:** number of bits
- **n:** max elements to store
- $ \epsilon \in [0, 1] $: Probability of false positive acceptable

So our constructor must take $ n $ and $ \epsilon $.

##### Intuition
- m is linearly propotional to n, so more elements you need to store, the more bits you need. <br/>
- m is dependent on $ -\log \epsilon $ the more accuracy you want, the required bits will shoot up. Use a graph plotter.

```rust
impl BloomFilterProd {
    //       v--- constructor convention in rust, static function (new not required)
    pub fn new(elements: usize, false_probability: f32) -> Self {
        let log2 = 2f32.ln(); // log(2)

        // m = -n * log2(p) / ln(2)
        let bit_count = -((elements as f32 * false_probability.log2()) / log2).ceil() as usize;
        // k = m/n * ln(2)
        let hash_count = (bit_count as f32 / elements as f32 * log2).ceil() as usize;

        Self {
            bits: bitvec![0; bit_count],
            hash_count,
        } // no semicolon, return
    }

}
```

### hash function
`additive_hasher` was sub-par to insult in short. `seahash` provides a better hasher, we will be keeping our wrapper so we can replace it in future also it's easier to use this way.
```rust
impl BloomFilterProd {
    fn hash(&self, key: &str, seed: usize) -> usize {
        let hash = seahash::hash_seeded(key.as_bytes(), seed as u64, 0, 0, 0) as usize;
        hash % self.bits.len() // get an index
    }
}
```

#### optimum hash function count
The fewer hash functions you have, the more is the probability of two different inputs having the same hash so too few hash functions are not good. Increasing the number of hash functions activates more bits, but having too few bits relative to the number of hash functions can lead to nearly all bits being activated, which is not good.

Formula for optimum hash count:

$$
k = \frac{m}{n} \log 2
$$

<details>
  <summary>Unimportant Sidenote</summary>

  If you put the above derived value of **m**, meaning if you chose bits correctly hash functions count only depends on acceptable `false_probability`.
  $$
  k = -\frac{\log_2\epsilon}{\log2^2}
$$
</details>

These changes are already incorporated in constructor.

### impl BloomFilter

```rust
impl BloomFilter for BloomFilterProd {
    fn insert(&mut self, key: &str) {
        //        v--- range in rust(easy for loops)
        for i in 0..self.hash_count {
            let hash = self.hash(key, i);
            //                        ^--- seed
            self.bits.set(hash, true)
        }
    }

    
    fn contains(&self, key: &str) -> bool {
        //                    v--- return true if all invocations return true
        (0..self.hash_count).all(|i| {
            let hash = self.hash(key, i);
            self.bits[hash]
        })
    }
}
```

### Usage
```rust
// ...imports
fn main() {
    let mut bl = BloomFilterProd::new(10, 0.01);
    // let mut bl = BloomFilter32::default();

    bloomy(&mut bl);
}

// ...fn bloomy
```
That's it, one line changed. Power of interfaces.
```
+-----------+-------+
| carrot    | false |
+-----------+-------+
| radish    | false |
+-----------+-------+
| vegetable | false |
+-----------+-------+
| onion     | false |
+-----------+-------+
| mango     | true  |
+-----------+-------+
| apple     | true  |
+-----------+-------+
| orange    | true  |
+-----------+-------+
| banana    | true  |
+-----------+-------+
```

## Conclusion
We looked into what is a bloom filter and how to go around implementing one. We looked into intuition of what factors(number of elements, acceptable false probability) determine BloomFilter implementation(number of bits, hash_function count). Bloom filters save a lot of resources around the world look into them, and maybe use them.
