# Magic Forest Benchmarks
[![build status](https://secure.travis-ci.org/clux/magic-forest.svg)](http://travis-ci.org/clux/magic-forest)

Personal comparisons of languages based on what I consider idiomatic use of these languages. Don't read too much into these.

This is based on the old blog post [Fast Functional Goats, Lions and Wolves](http://unriskinsight.blogspot.co.uk/2014/06/fast-functional-goats-lions-and-wolves.html) which gathered some publicity in 2014, but was broken by people finding an analytical solution.

## Rules
Only the brute forcing solution is used for all languages.

- Build up the entire tree mutation by mutation from the initial forest
- Perform *all* the insertions into a vector or similar, *then* filter out invalids
- At the end of `mutate` remove duplicates (converting to a set here / sort + erase combo / unique function)
- Do not use sets *before* you have done inserts and filter in `mutate`
- Continue doing mutation steps until a stable solution is found
- No third party libraries
- Language idiomatic solution (no obscure/largely expanding optimizations)

No analytical solutions, nor optimized search paths will be employed. Perform the sanity check below:

## Sanity
To verify you are actually doing all the work, print every element in the array/vector in `mutate` before returning it (i.e. after sort + dedup steps). With the arguments `55 45 50` input, there should be an extra `28866` lines of output from debug prints (see [debug.diff](./debug.diff) and [test.sh](./test.sh)).

## Usage
Build and run all sequentially with [bench.sh](./bench.sh) or run them individually:

```bash
# python
time ./forest.py 305 295 300

# python (pypy)
time pypy forest.py 305 295 300

# node
time ./forest.js 305 295 300

# go
go build forest.go
time ./forest 305 295 300

# c++ (gcc)
g++ -O3 -std=c++14 forest.cpp -o cppforest
time ./cppforest 305 295 300

# c++ (llvm)
clang++ -O3 -std=c++14 forest.cpp -o cppforestllvm
time ./cppforestllvm 305 295 300

# rust
rustc -C opt-level=3 forest.rs
time ./forest 305 295 300

# elixir
time ./forest.ex 305 295 300

# haskell (ghc)
ghc -O3 forest.hs
time ./forest 305 295 300

# fortran (gcc)
fortran -O3 forest.f08 -o fortranforest
time ./fortranforest 305 295 300

# ruby
time ./forest.rb 305 295 300

# shell
time ./forest.sh 305 295 300

# scala
scalac -opt:_ forest.sc
time scala Main 305 295 300

# kotlin
kotlinc forest.kt
time kotlin ForestKt 305 295 300
```

## Personal Results
Last tested August 2017 on an i7 7700K, using latest packages in Arch: stable rust (1.19), python 3.6 and pypy 5.8, node 8.3, go 1.8, c++ with both llvm4 and gcc7, haskell with ghc8, elixir 1.5.0, ruby 2.4, kotlin 1.1.3, scala 2.12 (jre-1.8).

- rust: 295ms
- c++/gcc: 295ms
- c++/llvm: 310ms
- kotlin: 670ms
- fortran: 750ms
- go: 1.4s
- scala: 2.1s
- haskell: 2.7s
- python/pypy3: 3.9s
- elixir: 5.5s
- node: 6.7s
- ruby: 11s
- python/3: 16s
- shell: 35s

All results are based on the above input data `305 295 300`, where the exponential nature of the problem really highlights the differences between languages.

### Discussion
#### Haskell
A very clean solution, but takes more to reason about when it comes to performance; using `nub` and lists would use 4 minutes to solve this problem, but using `Data.Set` (which is sort of cheating), it would drop to 3s.

The culprit turns out to be primarily `nub` from `Data.List`. This has been explained in the [haskell-ordnub repo](https://github.com/nh2/haskell-ordnub).

Beyond this issue, the Haskell solution is its usual clean style. One of the shortest implementations by LOC even with having to reimplement `nub`.

It is perhaps a bit unidiomatic to re-implement a dedup algorithm, but it's sufficiently short and performance critical that we have to allow it in order to avoid completely ignoring Haskell.

#### Elixir
Another clean functional implementation, benchmarked using the elixir script runner. Very cool and enjoyable dynamic functional style.

Compiling it with `mix` was attempted using an `escript` key in a `mix.exs` file, but this yielded no performance benefits in this case.

#### Python
Varies greatly based on how you implement this. A `class Forest` with it's own `__eq__` and `__hash__` for `Set()` uniqueness is hugely slower than a shorter and more functional `namedtuple` solution (which you can just stuff in a `Set` by itself). Pypy will also optimize the worse versions (worse under default interpreter) better than it will optimize already optimized python versions.

Current solution is the shortest one (shorter than the shortest haskell one we've had), and performs solidly under pypy for an interpretted language.

Pre-allocation of the list in `mutate` turned out to be slightly faster under `python`, but somehow slower under `pypy`, go figure.

#### Ruby
Surprisingly performs better than default python even when using the class construct. Python had to migrate away from that to maintain some semblance of speed.

Using `to_s` method as `uniq` condition couldn't manage to make it use the equality operators in a sensible way. A concise `to_s` is therefore much more performant.

#### Node
Lands bang in the middle of the two python implementations. Solid effort for having to implement its own duplicate element filter.

It's also a clean functional solution. Historically native loops have been faster than stuff like `reduce`, but on node 6, using `forEach` with a correctly sized `new Array(3*forests.length)` as a starting point in `mutate` actually turned out to be quite detrimental to performance (14->18s).

Using `Set` in the `reduce` is an option, but this caused my node to dump core after using all my heap memory.

#### Scala
Pretty clean and concise solution as you'd expect from a modern language, but this still feels super weird to write in. Implicit parameters and sometimes missing parens really creep me out.

It performs decently for having compiled strict typing, but nothing to be particularly impressed by.

The `distinct` method on `ArrayBuffer` appears to be the only way to deduplicate. Sorting first just slows it down. Aey implemented ordering appears not to be used, so a sensible default is probably derived from the case class instruction.

#### Kotlin
Fundamentally equivalent semantically to the Scala solution, but performs 3x better.

Equality and a print implementation is auto-derived for a `data class` so there's fundamentally little to do. You would think that this makes it a good 1-1 relation with Scala's `case class` statements, but performance wise, I guess not.

There's a lot of very specific object orientation based syntax in the language that weirds me aout a bit, and and the collection constructors feels inconsistent between each other, but the language generally reads very nicely.

#### Go
Relatively straight-forward implementation straightened out by [@jas2701](https://github.com/jas2701). Converts into set equivalents (maps with comparable keys) near the end rather than using a sort dedup and probably suffers a bit for it.

The explicit filters functions having to be inlined everywhere makes this solution quite verbose, but it is faster than all but the super serious compiled languages.

#### C++
Modern cpp solution is very readable at only a few more lines than the scripting language counterparts. A bit of mental overhead on emplace, awkward iterator sempantics with back inserters, but even that is pretty standard modern cpp really.

It turns out that creating a new list with the unique iterator than erasing from it is faster, but we're trying to keep languages between semantics similar.

There's a more standalone function approach here than rust due to not having traits, and also that passing member functions can look awkward in long `<algorithm>` instructions.

The choice of `-std=c++11` vs `-std=c++14` made no difference in performance.

Deleting or adding the default constructor of forest actually gained 10% performance, we're not sure why.

gcc seems to be consistently around 5% faster than clang.

#### Fortran
A solution 3x longer than the dubious second place holder in LOC; `go`. It stays within the rules and implements its own quick sort, and as far as Fortran goes, it's remarkably understandable. It was graciously offered by [@jchildren](https://github.com/jchildren) in [#4](https://github.com/clux/magic-forest/pull/4).

It's up there in the top 3 languages, but it still performs worse than cpp/rust by a factor of two. It's certainly as close to the metal as these languages, so there should perhaps be room for improvement here without going too nuts.

#### Shell
Stream implementation using `awk` and `grep` because [@Thhethssmuz](https://github.com/Thhethssmuz) thought it was possible. Turns out it's not too terrible, it's actually pretty amazing that it works as well as it does considering the overhead of forking around 3000 processes on the normal input set.

 Slightly different semantics because we are working entirely with streams, but I'll have to allow it. It's the shortest solution here by far.

Interestingly the while condition is actually what prints the result to stdout so that serves as a dual "filter out unstable solution" and "end the while loop" at the same time, saving us any extra processing in solve after the while loop.

#### Rust
Rust impressively ~ties C++ with completely normal code. It all seems to come down to how many iterator operations you have to do. The original rust solution I saw online was not using `retain` and this saved quite a bit on performance.

Perhaps the least magic implementation of the bunch. It's not as nice or short as python / haskell, but at least you can reason about its performance.

Wrapping `forest.rs` inside a `cargo` built project provided no change in performance when building with `cargo build --release` despite more numerous compiler/linker flags.

#### A note on sets vs. sort + dedup
Both C++ and rust versions can be made somewhat shorter by having the `mutate` function return a set, in C++:

```cpp
set<Forest> mutate(const set<Forest> &curr) {
  vector<Forest> next;
  next.reserve(curr.size() * 3);
  for (auto f : curr) {
    next.emplace_back(f.goats - 1, f.wolves - 1, f.lions + 1);
    next.emplace_back(f.goats - 1, f.wolves + 1, f.lions - 1);
    next.emplace_back(f.goats + 1, f.wolves - 1, f.lions - 1);
  }
  auto valid_end = remove_if(begin(next), end(next), is_invalid);
  return set<Forest> s(next.begin(), valid_end);
}
```

and in rust:

```rust
fn mutate(forests: BTreeSet<Forest>) -> BTreeSet<Forest> {
    let mut next = Vec::with_capacity(forests.len() * 3);
    for x in forests.into_iter() {
        next.push(Forest::new(x.goats - 1, x.wolves - 1, x.lions + 1));
        next.push(Forest::new(x.goats - 1, x.wolves + 1, x.lions - 1));
        next.push(Forest::new(x.goats + 1, x.wolves - 1, x.lions - 1));
    }
    next.into_iter().filter(|x| x.is_valid()).collect()
}
```

Both of these still obey the rules (and some languages implement this strategy), but quick investigation reveals that to be more than twice as expensive as just using a sort into a linear erase combo.
