# We have linear types at home
I woke up early the other day and ended up reading "Without boats" newest Rust article: [References are like jumps](https://without.boats/blog/references-are-like-jumps/) (great article btw). It got me thinking about one of his other blog posts, the "do ... final" one ([Asynchronous clean-up](https://without.boats/blog/asynchronous-clean-up/)), and more generally about linear types themselves. My mind drifted to the blog posts by Niko Matsakis made on the same topic: [Must move types](https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/) & [Unwind considered harmful?](https://smallcultfollowing.com/babysteps/blog/2024/05/02/unwind-considered-harmful/).

And then my brain connected the dots: I can probably get something working in Rust _right now_.

## What's a linear type anyways
I'll just give a brief description of what I mean when I use the term "linear type" (if you're already familiar with them you can skip this section). Objects of linear types have a simple property: They're only used once. This means that the objects are created at some point using a _constructor_, and then at a later point they are destroyed by a _destructor_. There might be many available constructors and destructors, but a single object uses exactly one of each. This separates them slightly from normal Rust types (which are often referred to as _affine types_): Those only have a single destructor, which might not even run as objects can be leaked. 

Let's do an example with a file-type, to get a feel for this. Assume we have a type called `File`, with a single constructor `open`, and a single destuctor `close`. We ignore the implementations:

```
pub struct File { ... }

impl File {
    pub fn open(p: &std::path::Path) -> Result<File, std::io::Error> { ... }
    pub fn close(self) -> Result<(), std::io::Error { ... }
}
```

The following code would then fail to compile, as the object `f` is created, but not destroyed:

```
fn do_things_with_file(path: &std::path::Path) -> Result<(), std::io::Error> {
    let f = File::open(path)?;

    // ...

    Ok(())
}
```

However, this on the other hand would succeed:

```
fn do_things_with_file(path: &std::path::Path) -> Result<(), std::io::Error> {
    let f = File::open(path)?;

    // ...

    // remember to close the file!
    f.close()?;

    Ok(())
}
```

An important point to make is that _all paths_ need to close the file, so if you return early you have to remember to close the file in those paths as well:


```
use std::io::Read;

fn read_to_string(path: &std::path::Path) -> Result<String, std::io::Error> {
    let mut f = File::open(path)?;

    let mut buf = String::new();
    if let Err(e) = f.read_to_string(&mut buf) {
        // We have to close the file here as well, but we'll just ignore the result
        let _ = f.close();

        // Now we can safely return!
        return Err(e);
    }

    // remember to close the file!
    f.close()?;

    Ok(buf)
}
```

## Maybe the real linear types were the const we got along the way
So the basic idea is to (ab)use the "soon-to-be-stabilized" `const`-blocks. If you pay close attention to the development of Rust you might know that this is coming to Rust on the next stable release, so you can test it today on beta if you want to try this yourselves. The idea of `const`-blocks is to allow guaranteed compile-time execution, so you can very soon write the following code on stable Rust:

```
let a = const { 4 + 3 };
const { assert!(6 < 13) };
```

You probably want to do something cooler than math (although math is pretty dope tbh), but you get the gist. It's a very nice feature. However, we can abuse this. `const`-asserts are a nice way of saying "give me a compilation error if this happens", so let's stick it in a `drop` to say "if this object is ever dropped, then give me a compilation error".

```
struct BadLinearType;

impl Drop for BadLinearType {
    fn drop(&mut self) {
        const { assert!(false, "Don't drop this!") };
    }
}
```

Sadly, this naïve implementation fails to compile the drop itself, which is not what we wanted. We want it to fail at every _callsite_ to drop. We can get to this by using another `const` concept: `const`-generics.

```
struct LinearType<const DROP: bool = false>;

impl<const DROP: bool> Drop for LinearType<DROP> {
    fn drop(&mut self) {
        const { assert!(DROP, "Don't drop this!") };
    }
}
```

The `DROP` constant will always be false, so the assert will fail at every call to drop. But, according to the compiler there is nothing inherently wrong with the implementation of drop itself, so _it_ will still compile. Success! [^1]

[^1]: This seems to also work with `const { assert!(false) }`, it appears that the mere presence of the `const`-generic is enough. I assume this is because the error is pushed to each monomorphized version of `drop`, so it needs atleast one callsite to error out.

Okay, so we have a basic form of linear types in Rust, nice! A couple of bummers:
1. We need to run `cargo build` in order to get this error, as the call to drop is not inserted in `cargo check`.
2. The compiler messages are abysmal: They only point to the `drop` implementation, never to the callsite.
3. A user can still leak objects of these linear types (e.g. via `std::mem::forget`), as that will also end up never calling `drop`.

## Everything can panic!
Our linear type is also very boring: It can be created, but never destroyed. Let's implement an API for a new, linear file type with a fallible drop call. You know, to see what it feels like to actually try and do something (somewhat) useful with this technique. What I quickly noticed was the problem of panics. These insert calls to `drop` everywhere! Simply calling a function, any function, in a non-`const` context caused a compilation error. So if we want to do cleanup we need to only use very primitve operations. So something like this won't work:

```
pub struct File<const DROP: bool = false> {
    file: Option<std::fs::File>,
}

impl File {
    pub fn open(path: impl AsRef<std::path::Path>) -> Result<Self, std::io::Error> {
        let file = std::fs::File::open(path)?;
        Ok(Self { file: Some(file) })
    }

    pub fn close(mut self) -> Result<(), std::io::Error> {
        // We can't call non-const functions here, so `Option::take` will fail
        let Some(_f) = self.file.take() else {
            // SAFETY: We only take the file in this function
            unsafe { std::hint::unreachable_unchecked() };
        };

        // We need to forget the object so it doesn't call drop!
        std::mem::forget(self);

        // We can do anything here though
        todo!("Cleanup file");
    } 
}

impl<const DROP: bool> Drop for File<DROP> {
    fn drop(&mut self) {
        const { assert!(DROP, "Don't drop this!") };
    }
}
```

My best way to go about this was to use raw pointers, as they can be copied, which is something the compiler knows cannot panic:

```
use crate::fs::File;

fn main() {
    // Construct our linear file type
    let file = File::open("my-file.txt").unwrap();

    // If we remove this line it will fail to compile
    file.close().unwrap();
}

mod fs {
    pub struct File<const DROP: bool = false> {
        // Heap allocated file object
        file: *mut std::fs::File,
    }

    impl File {
        pub fn open(path: impl AsRef<std::path::Path>) -> Result<Self, std::io::Error> {
            let file = std::fs::File::open(path)?;
            let file = Box::into_raw(Box::new(file));
            Ok(Self { file })
        }

        pub fn close(self) -> Result<(), std::io::Error> {
            // We can copy a raw pointer!
            let ptr = self.file;

            // We need to forget the object so it doesn't call drop!
            std::mem::forget(self);

            // SAFETY: We kept this alive, and we know that it's heap-allocated
            let _f = unsafe { Box::from_raw(ptr) };

            // Now we can proper cleanup
            todo!("Cleanup file");
        } 
    }

    impl<const DROP: bool> Drop for File<DROP> {
        fn drop(&mut self) {
            const { assert!(DROP, "Don't drop this!") };
        }
    }
}
```

This works! The file object now needs to be cleaned up up with the `File::close` method through all possible execution paths. However, the problem we solved for `File::close` still persists in the user's code. You can't call anything that can panic (from the compiler's perspective) between `File::open` and `File::close`. So all-in-all this linear type is not very useful. 

But wait... It's not really the panic that's holding us back, it's the unwinding!

## Abort!
Normally when a panic occurs in Rust the program will start to "unwind", meaning it will go (in reverse) over the current stack and clean up anything it comes across by calling `drop` on each varaiable. This is causing us a lot of trouble, but luckily, we can turn it off! We can set panics to abort instead of unwind. This means that our program will just hard exit on a panic, instead of trying to clean up neatly after itself. For a lot of programs we don't actually need the cleanup, so we can just as well turn it off here, and explore what's possible in this new space. Putting the following in our `Cargo.toml` will set all panics to abort:
 
```
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

We are now free to do anything while our linear type is alive, provided that all normal execution paths (including early returns) clean up the file. This also allows us to not do the ugly pointer shenanigans we needed in the previous section, we can go back to putting our `std::fs::File` in an `Option`, like normal rustaceans. The following is a somewhat more complete implementation of this, along with a `main`-function using the linear type for some basic operations:

```
use std::io::Read;

use anyhow::Context;

use crate::fs::File;

fn main() -> Result<(), anyhow::Error> {
    let mut file = File::open("file.txt").context("failed to open file")?;

    // try blocks would be really nice here
    let result_io = (|| -> Result<String, anyhow::Error> {
        // Let's get the metadata of the file...
        let metadata = file.metadata().context("failed to get metadata")?;

        // ... and print some shit!
        println!("Last modified: {:?}", metadata.modified()?);
        println!("Last accessed: {:?}", metadata.accessed()?);

        // Let's also grab all the content in the file
        let mut buf = String::new();
        file.read_to_string(&mut buf).context("failed to read to string")?;
        Ok(buf)
    })();

    let result_close = file.close().context("failed to close file");

    // Handle each possible case
    match (result_io, result_close) {
        (Ok(content), Ok(())) => {
            println!("File content:\n{content}");
            Ok(())
        }

        (Ok(_), Err(e)) | (Err(e), Ok(_)) => Err(e),

        (Err(io), Err(close)) => Err(anyhow::anyhow!(
            "Encountered multiple errors:\n{io}\n{close}"
        )),
    }
}

mod fs {
    use std::io::{Read, Write};

    pub struct File<const DROP: bool = false> {
        file: Option<std::fs::File>,
    }

    impl File {
        pub fn open(path: impl AsRef<std::path::Path>) -> Result<Self, std::io::Error> {
            let file = std::fs::File::open(path)?;
            Ok(Self { file: Some(file) })
        }

        pub fn metadata(&self) -> Result<std::fs::Metadata, std::io::Error> {
            self.file.as_ref().unwrap().metadata()
        }

        pub fn close(mut self) -> Result<(), std::io::Error> {
            let mut file = self.file.take().unwrap();

            // We need to forget the object so it doesn't call drop!
            std::mem::forget(self);

            file.flush()?;
            file.sync_all()?;

            Ok(())
        }
    }

    impl<const DROP: bool> Drop for File<DROP> {
        fn drop(&mut self) {
            const { assert!(DROP, "Don't drop this!") };
        }
    }

    impl Read for File {
        fn read(&mut self, buf: &mut [u8]) -> Result<usize, std::io::Error> {
            self.file.as_mut().unwrap().read(buf)
        }
    }
}
```

## What now?
Well, first of all you probably don't want to do this in production code. I mentioned earlier that the error messages are abysmal, and they truly are. On my computer I get the following error message (`rustc 1.79.0-beta.4 (a26981974 2024-05-10)`):

```
error[E0080]: evaluation of `<fs::File as std::ops::Drop>::drop::{constant#0}` failed
  --> src/main.rs:69:21
   |
69 |             const { assert!(DROP, "Don't drop this!") };
   |                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the evaluated program panicked at 'Don't drop this!', src/main.rs:69:21
   |
   = note: this error originates in the macro `$crate::panic::panic_2021` which comes from the expansion of the macro `assert` (in Nightly builds, run with -Z macro-backtrace for more info)

note: erroneous constant encountered
  --> src/main.rs:69:13
   |
69 |             const { assert!(DROP, "Don't drop this!") };
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

note: the above error was encountered while instantiating `fn <fs::File as std::ops::Drop>::drop`
   --> /Users/skogseth/.rustup/toolchains/beta-x86_64-apple-darwin/lib/rustlib/src/rust/library/core/src/ptr/mod.rs:514:1
    |
514 | pub unsafe fn drop_in_place<T: ?Sized>(to_drop: *mut T) {
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0080`.
```

It only mentions the implementation for the drop, not where it's called, and so I imagine chasing down bugs for this to be horrible. I also can't get it working with `async`/`.await`, but that might just be my lacking knowledge of that part of Rust standing in the way. 

I also wanted to mention that (despite the many problems with this technique) my experience using linear types this way has been very positive. I'm certain that a more well-supported version of this, with some proper error-messages, would be nice to use. Atleast in the case of "aborting panics", I'm still pretty sceptical of their use with "unwinding panics". I'm sure they'd be much nicer to use with the hypothetical `do ... final`, but I do get a feeling that handling linear types in unwinding paths would be very frustrating.
