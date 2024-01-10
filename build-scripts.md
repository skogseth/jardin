# Build scripts

Some general notes:
- Error handling is different. Use `unwrap` when prototyping, but switch to `expect` when you want good error messages!
- Paths are hard to handle correctly when you need to print them a lot. Either go all in on `camino` and use UTF-8 paths, or suck it up and expect to have a lot more annoyances related to error handling. And watch out for the footguns (pew pew) connected to this, I'll get back to those.

## How to best handle paths
There are two good choices for paths in Rust, depending on what you want to support:
1. The standard library paths `std::path::{Path, PathBuf}` (based on `OsString`s).
2. UTF-8 paths from the camino library `camino::{Utf8Path, Utf8PathBuf}` (based on `String`s).

Normally, I always go with the path types in `std`, which seems to also be the general consensus in the Rust community. The extra effort needed to sometimes handle `OsString`s is okay given the convenience of using the standard library combined with the advantage that your application/library will be able to handle some of the input from the operating system not being valid UTF-8. But emphasis on _some_, because for build scripts the scales start to tip...

Build scripts need paths to be valid UTF-8 to a much larger degree than most applications. Let's look at a problematic piece of code that may seem innocent:

```rust
use std::path::{Path, PathBuf};

fn main() {
    let source_dir = PathBuf::from(std::env::var("SOURCE_DIR").unwrap());
    let artifacts_dir = build_something(&source_dir);
    println!("cargo:rustc-link-search={}", artifacts_dir.display());
}

fn build_something(out_dir: &Path) -> PathBuf { ... }
```

This code has some problematic bugs. The core problem is that we are jumping back-and-forth between `OsString`s (in the form of std-paths) and regular `String`s. Let's first take a look at fixing this code using std-paths:

The first fix is for the call to `std::env::var` function. This function reads the value of the environment variable (if it's present) which yields an `OsString`. It then fallibly converts this to a `String`. We want to create a `PathBuf`, so we are perfectly fine with just the `OsString`, so the right function to call is `std::env::var_os`. Note: Another cool part of the `std::env` module is the `vars`/`vars_os`-functions, which yield iterators over _all_ defined environment variables (again as either `String`s or `OsString`s).

 A bigger problem is however the last line, which uses the `display`-method on `Path`. The `display` method is, as the name suggests, only meant to display paths to the user. The method attempts to interpret the path as UTF-8, and will replace invalid bytes with the unicode "unknown symbol": �. It is therefore better to fallibly convert it to UTF-8, as this will emit an error if it can't be converted. The `display` method will silently print a string that is not the actual path you want to tell cargo about. Note: It seems like cargo ignores invalid UTF-8 in the output from build scripts, so you can't solve this by printing the bytes directly to `stdout`.

In this case the information emitted to cargo is critical, but the error still won't show up until the linking of the final binary fails (as rustc will then fail to find whatever artifact you placed in that directory). For cargo emits such as `println!("cargo:rerun-if-changed={...}")` the information is less critical, as the bug will only make the build-script not rerun under the right conditions. However, one could argue that this bug is even worse as it will go under the radar and only cause problems way down the road.

A fixed version of the code is found below.

```rust
use std::path::{Path, PathBuf};

fn main() {
    // Use `std::env::var_os` to not require the value to be valid UTF-8
    let source_dir = PathBuf::from(std::env::var_os("SOURCE_DIR").unwrap());

    // No changes, all good, 10/10 would call this function again!
    let artifacts_dir = build_something(&source_dir);

    // Check for valid UTF-8, don't just "display" the path
    println!("cargo:rustc-link-search={}", artifacts_dir.to_str().unwrap());
}

fn build_something(out_dir: &Path) -> PathBuf { ... }
```

Let's also look at how to handle this with camino's UTF-8 paths. This case is a lot simpler, we replace the types from `std` with the equivalent types from `camino`. The call to `std::env::var` is now correct, and the camino-paths implement display so we can simply print them.

```rust
use camino::{Utf8Path, Utf8PathBuf};

fn main() {
    // It is now correct to use `std::env::var` as we want the variable to be valid UTF-8.
    let source_dir = Utf8PathBuf::from(std::env::var("SOURCE_DIR").unwrap());

    // Another banger of a line, wow! 7 stars!
    let artifacts_dir = build_something(&source_dir);

    // The path is already guaranteed to be valid UTF-8, so we can simply print it
    println!("cargo:rustc-link-search={artifacts_dir}");
}

fn build_something(out_dir: &Utf8Path) -> Utf8PathBuf { ... }
```

The code in this case is a little simpler, which I have found to _often_ be the case with UTF-8 paths in build scripts. The path is checked to be valid UTF-8 once (in the `std::env::var` function), and so we don't have to worry about it any more. The advantage is small here, but when you find yourself constantly emmiting paths to cargo it's pretty nice. 

One big disadvantage of using `camino` is the interaction with the rest of the Rust ecosystem, as that have (for good reasons) converged around using the std-paths. Things that accept paths are still nice to use, as `Utf8Path` implements `AsRef<Path>`. However, things that return paths now need to be validated to be UTF-8. There are UTF-8 equivalents for a lot of the annoying parts of `std` in the camino library, to help mitigate this, but for external libraries this is not the case. One particular case of this was when using `camino` together with `dunce`. 

## Lessons learned
Old code, with multiple types of bugs. A big one is that the whitespace separating args is not supported by all terminals, altough I think you're mostly good for Windows/macOS/Linux.

```rust
fn define(key: impl Display, val: impl Display) -> String {
    format!("-D {key}={val}")
}
 
let output = Command::new("cmake")
    .arg(root)
    .arg("-G Ninja")
    .arg(format!("-B {}", build_path.display()))
    .arg(define("CMAKE_INSTALL_PREFIX", install_path.display()))
    .arg(define("CMAKE_EXPORT_COMPILE_COMMANDS", "ON"))
    ...
```

Fixing the whitespace issue and using `OsString`s where appropriate:

```rust
fn define(key: impl AsRef<OsStr>, val: impl AsRef<OsStr>) -> [OsString; 2] {
    [
        OsString::from("-D"),
        OsString::from_iter([key.as_ref(), OsStr::new("="), val.as_ref()]),
    ]
}

let output = Command::new("cmake")
    .arg(root)
    .args(["-G", "Ninja"])
    .args([OsStr::new("-B"), build_path.as_os_str()])
    .args(define("CMAKE_INSTALL_PREFIX", &install_path))
    .args(define("CMAKE_EXPORT_COMPILE_COMMANDS", "ON"))
    ...
```
