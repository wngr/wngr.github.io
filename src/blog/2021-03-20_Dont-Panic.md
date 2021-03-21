# Dont `panic!()`
The general error handling story with Rust feels very ergonomic and more importantly very obvious and plausible to a
programmer.  There are exactly two kind of errors:
1. Recoverable errors, represented by the `Result<T,E>` type, and
1. unrecoverable errors, where the program panics.

This post is about the latter: What happens exactly if we call `panic!()` in a Rust program, which options are there to
modify and influence this behaviour, and what are the caveats? This write-up is non-exhaustive, but highlights the
noteworthy points that helped me understand better. Platform specific nuances are not considered.

Panics are commonly used to enforce an invariant of a program, which must not be broken and is completely unexpected by
the application's logic. For further considerations when to use panics or not, see this
[section](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) in the Rust book.

Let's consider a simple panic[^0]:
```rust,should_panic
panic!("boom");
println!("Nobody prints me :-(");
```
The argument passed to the `panic!` macro is printed, and further execution is stopped. In the following, we will trace
what just happened.

### The `panic!` macro
The panic macro is defined in [std](https://doc.rust-lang.org/src/std/macros.rs.html#12-18):
```rust,no_run,noplayground,ignore
macro_rules! panic {
    () => ({ $crate::panic!("explicit panic") });
    ($msg:expr $(,)?) => ({ $crate::rt::begin_panic($msg) });
    ($fmt:expr, $($arg:tt)+) => ({
        $crate::rt::begin_panic_fmt(&$crate::format_args!($fmt, $($arg)+))
    });
}
```
So it either accepts a string literal with formatting args (like `format!`), or an object. The object needs to fulfill
the `Any + Send` trait bounds to be able to call
[`begin_panic`](https://github.com/rust-lang/rust/blob/507bff92fadf1f25a830da5065a5a87113345163/library/std/src/panicking.rs#L513-L521):
```rust,no_run,noplayground,ignore
pub fn begin_panic<M: Any + Send>(msg: M) -> ! {
    if cfg!(feature = "panic_immediate_abort") {
        intrinsics::abort()
    }

    let loc = Location::caller();
    return crate::sys_common::backtrace::__rust_end_short_backtrace(move || {
        rust_panic_with_hook(&mut PanicPayload::new(msg), None, loc)
    });
```
Interesting, with the `panic_immediate_abort` feature set, the program immediately aborts without any further actions.
This is not to be confused with the `panic = "abort"` setting in `Cargo.toml`, which sets the panic abort runtime. This
is used in order to get really tiny binaries. You need to
[compile](https://github.com/rust-lang/rust/issues/54981#issuecomment-443369450) Rust's std library yourself with said 
feature flag set in order to use it.

### Panic hook
`rust_panic_with_hook` will do some checking for recursive panics (it will terminate immediately in this case), and
execute the panic hook, before dispatching to the panic runtime to unwind. Users can set a custom panic hook, which is
executed for every panic, with `std::panic::set_hook`. This is a global resource, so last writer wins.
```rust,should_panic
std::panic::set_hook(Box::new(|_panic_info| {
  println!("Hello Panic!");
}));
panic!();
```
The `PanicInfo` type propagates meta information of the panic (location, payload or message; more on that later). Any
custom set panic hooks can be unset by `std::panic::take_hook()` which will leave Rust's default panic hook in place.

### Panic runtime
After the panic hook has been executed, a FFI function[^1] is called:
```rust,no_run,noplayground,ignore
fn __rust_start_panic(payload: *mut &mut dyn BoxMeUp) -> u32;
```
This is a native interface, because rustc will make sure that user chosen panic runtime is linked against after
compilation.

Rust provides two panic runtimes: `panic_abort` and `panic_unwind`. `panic_abort` just aborts the program (via a
platform specific syscall, e.g. `libc::abort()`) and can be enabled in `Cargo.toml`:
```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

The default is
[`panic_unwind`](https://github.com/rust-lang/rust/blob/89882388d931d2e4d0d30c73fc1aa9c56f4df110/library/panic_unwind/src/lib.rs#L1-L12),
which implements platform specific stack unwinding: In general, an unwinder walks the stack from top to bottom, where
for each frame a so-called "personality routine" is called. This routine determines for the individual stack frame,
which actions to take to handle an exception. If a handler frame has been found (so basically a catch block), the
cleanup phase begins, where for each stack frame a "landing pad" is generated, which runs destructors, frees memory etc.
Once the stack has been unwound, control is transferred to the handler frame.  If no handler frame is found, the process
is terminated. This is not unique to Rust, but behaviour of the [LLVM
backend](https://llvm.org/docs/ExceptionHandling.html#overview).

### Catching a panic
With the `panic_unwind` runtime, a panic in a thread triggers the unwinding of the stack. After the stack has been
unwound, and no handler frame (catch block, see above) has been found, the process is terminated.  The Rust standard
library provides a mechanism to "catch" a panic after the stack has been unwound: `std::panic::catch_unwind`, which takes
a closure that might potentially panic, and returns a `Result<T, Box<dyn Any + Send + 'static>>`. This eventually ends
up in the compiler intrinsics, where LLVM instructions for a try-catch block are fabricated (for GNU, see
[here](https://github.com/rust-lang/rust/blob/master/compiler/rustc_codegen_llvm/src/intrinsic.rs#L537-L542)). With
that, `std::panic::catch_unwind` is able to catch a panic, after the stack has been safely unwound.

But why even try to catch a panic at all, given that it's considered a unrecoverable error? There are a couple of
potential motivations: supervision of third party (library) code, robustness concerns (think safety critical systems),
or managing of thread pools (for example in async runtimes). The canonical example of this is
[`std::thread`](https://doc.rust-lang.org/src/std/thread/mod.rs.html#473-475): Any panics encountered in a spawned
thread are unwound, after which the (OS) thread exits. The `JoinHandle` returns from the thread's `spawn` method will
yield the panic, if any happened. `tokio::task::spawn` took a similar approach (sans the OS thread exit, of course).

`std::panic::resume_unwind` provides a way to trigger a panic without invoking the panic hook. If invoked with a panic
caught by `std::panic::catch_unwind`, the panic hook has been already executed for said panic, and the stack unwound.
The stack will be unwound again with `resume_unwind`, but this time without any catch blocks (if not nested with another
`catch_unwind`).

Note that all types passed into `catch_unwind` need to be
[`UnwindSafe`](https://doc.rust-lang.org/std/panic/trait.UnwindSafe.html). When used only from within safe Rust code,
problematic is only shared mutable state, which could violate application level invariants.

### Reacting to panics in application level code
Writing applications in Rust, I think it's advisable to stick to the recommendations of the Rust core team regarding
error handling[^2]. Panics should be used as a last resort for non-recoverable errors. The logical conclusion from that
is to always use the `panic_abort` runtime, so that any panic in any thread will bring down the whole process. However,
it's advisable to have some sort of shutdown procedure (close database connections, flush to disk, etc.). This can be
implemented by placing a custom panic hook with `std::panic::set_hook`[^3]. As long as the closure passed to the hook
finishes all work synchronously, we can stick with the `panic_abort` runtime; if we can only signal the shutdown to
other threads and then yield, we obviously have to use the `panic_unwind` runtime -- otherwise the program would
immediately exit after the panic hook returns.

----

#### Footnotes

[^0]: This is a runnable snippet, so you can click on the play button above.

[^1]: If you're curious about the type of `payload`, check out [this
explanation](https://rustc-dev-guide.rust-lang.org/panic-implementation.html#std-implementation-of-panic).

[^2]: The Rust Book has a very well written [chapter](https://doc.rust-lang.org/book/ch09-00-error-handling.html) on error
handling, which is worth a read.

[^3]: For some inspiration to parse `PanicInfo`, check out the [default hook in
std](https://github.com/rust-lang/rust/blob/507bff92fadf1f25a830da5065a5a87113345163/library/std/src/panicking.rs#L180-L227).
