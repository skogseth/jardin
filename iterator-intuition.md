# Iterator intuition

## Background
This was inspired by a 2019 talk by Conor Hoekstra called "Algorithm intuition". This talk was in turn inspired by a 2013 talk by Sean Parent called "C++ Seasoning" which talked a lot about rotations (the array kind, not the SO(3) kind), but more importantly emphasized a need for people to learn the algorithms included in the C++ standard library. Sean gave this talk because he in turn had been inspired by the way Alex Stepanov wrote code, which for the uninformed is the father of STL (Standard Template Library) which became the C++ standard library before it's standardization in (I wanna say) 1998.

So why? Why do these people, now including me, talk about learning these standard algorithms? And what do I (and Conor Hoekstra) talk about when we say "intuition"? Well, it's largely about writing good code, by which I mean:

- Readable
- Performant
- Reliable

(which I now realize is pretty close to the "reliability / performance / productivity" motto of Rust).

When you use algorithms, and in this talk specifically those associated with Rust's iterators, you can more easily achieve these. Readable is often beholden to the eye, but more importantly it is something you can train. If you, and others, learn standard algorithms you start to develop a language for communicating programmer intent. This also causes the two other points. When you accurately communicate the intent of your code in the code itself it becomes more readable (as the intent is actually written out), more performant (as you have more accurately told the compiler the constraints of your solution, which can be used for optimizations) and more reliable (as you will avoid mismatches between what you meant to write and what you actually wrote). I'll try to get into these points more if there is time (lines?).

## The `Iterator` trait
The `Iterator` trait is defined as something like:
```rust
pub trait Iterator {
  type Item;

  fn next(&mut self) -> Option<Self::Item>;
}
```

(there are more methods as I am sure you now, we're getting there).

Most are probably familiar with this, but it's worth noting and thinking about the exact concept that this trait is encapsulating, because it's quite limited. It only encapsulates the notion of "something that _may_ yield something when asked to". This is very different from, for example, C++'s iterators, as they are better described as "something that you can move within". C++'s iterators are more powerful in some ways, being able to implement many algorithms that are far beyond the scope of Rust's iterator trait, but they suffer in other ways. They are known for being notoriously difficult to implement. They also can't express things that may yield things only once, or iterators that can only be iterated over one way.

## The basics

```rust
fn odds_squared(list: Vec<i32>) -> Vec<i32> {
    list.into_iter()
        .filter(|x| x % 2 != 0)
        .map(|x| x * x)
        .collect()
}
```

## Creating iterators
String:
- bytes
- chars

Most collections (Vec, HashSet, HashMap, etc.):
- into_iter
- iter/iter_mut

Slices:
- windows
- chunks
- chunks_by (hopefully soon)

HashMap / BTreeMap
- keys
- values

Free standing
- std::env::vars/std::env::vars_os
- std::fs::read_dir

### Examples
Taken from a build-script:
```rust
fn valid_env_vars(substr: &str) -> Vec<String> {
    std::env::vars_os()
        .filter_map(|(key, _)| key.into_string().ok())
        .filter(|s| s.contains(substr))
        .collect()
} 
```

Quite involved example around reading a directory
```rust
fn get_executables(path: &Path) -> Result<Vec<PathBuf>, anyhow::Error> {
    std::env::read_dir(path)
        .context("Directory with executables not found")?
        .filter_map(Result::ok) // <=> |result| result.ok()
        .map(|entry| entry.path())
        .filter(|path| {
            path.extension()
                .is_some_and(|ext| ext == OsStr::new(".exe"))
        })
        .collect()
}
```

## Consumers
Four types:
- 0 => for_each
- 0/1 => reduce
- 1 => fold
- n => collect

Lots of helper methods fall into either reduce or fold, which btw are just Rust terms I'll use here, both of these names are used to describe the same concept depending on your language (e.g. C++ uses reduce whilst Haskell uses fold). Only Rust, as far as I am aware, uses both to describe slightly different algorithms.

Reductions (in Rust) means taking an iterator and reducing all of it's elements down into one based on some function. But, this means that if the iterator has no elements, that there will be no result. So reduce, along with its friends, yield an `Option<T>`. There are only a few "friend functions" that are similar to reduce.
- min/max and their siblings *_by and *_by_key (which btw are best friends with the siblings of Vec::sort)
- find (find_map seems weird, look into it)
- position (??)

Folds (again in Rust) means taking an accumulator value and folding all the values of the iterator into this based on some function. This differs from the reduce in that it always yields a value, if the iterator was empty then the yielded value is simply the starting value of the accumulator. Its friends include
- sum/product
- count
- all/any
- etc.

## Changes to the iteration
- take/take_while
- skip/skip_while
- step_by
- intersperse
- zip

## The black sheep: scan
This algorithm is a bit of a monstrosity, but it can be very powerful.

Problem: Find the total water collected in the mountain.
```rust
fn water_collected(list: &[u32]) -> u32 {
    list.into_iter().scan(0, |max, current| {
        *max = std::cmp::max(*max, *current);
        Some(*max - *current)
    }).sum()
}
```

## MORE METHODS
- Lots of methods named something like try_*, which works with fallible types (although it's good to remember that `Result` implements `FromIterator` so you can collect into a `Result`)
- DoubleEndedIterator => `rfold` and friends
- Itertools (from the itertools crate)
- Step (which seems to be a "step" in the direction of C++ style iterators)
