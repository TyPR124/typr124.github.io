# Some Nuances of Undefined Behavior in Rust

I recently came across an interesting FFI situation, which I wanted to understand a bit better. Now I'd like to share what I found in the hopes of making everyone a better programmer.

This post is about undefined behavior, or UB. Specifically UB in Rust at the language level. This basically means we're going to do something that the compiler is free to assume we never did. That sounds bad, right? How can the compiler just assume we didn't write the code that we wrote? Well, it is as bad as it sounds, even if it doesn't always cause bad things to happen.

There are many things that constitute UB in Rust. None of these things can be done without writing some `unsafe` code. But we as programmers want to get the most out of our language, and sometimes that means writing some `unsafe` code. As great a language as Rust is, it does not make writing bad code impossible. Let's take a look at one such bit of code.

```rust
fn ptsname_r(fd: &PtyMaster) -> nix::Result<String> {
    use std::ffi::CStr;
    use std::os::unix::io::AsRawFd;
    use nix::libc::{ioctl, TIOCPTYGNAME};

    // the buffer size on OSX is 128, defined by sys/ttycom.h
    let buf: [i8; 128] = [0; 128];

    unsafe {
        match ioctl(fd.as_raw_fd(), TIOCPTYGNAME as u64, &buf) {
            0 => {
                let res = CStr::from_ptr(buf.as_ptr()).to_string_lossy().into_owned();
                Ok(res)
            }
            _ => Err(nix::Error::last()),
        }
    }
}
```

This is an implementation of the `ptsname_r` function for Mac OSX. Can you spot the UB?

The problem here is that we create an immutable buffer, but then mutate it in the call to `ioctl`. At this point, I'm pretty sure I've spotted UB, but my curiosity demands I look closer.

Before looking any closer though, I do want to point out the fix. It is simple enough, just make `buf` mutable, and in our call to `ioctl`, pass `&mut buf` instead of `&buf`. Problem solved. And even if this isn't *technically* UB, you can see that this is the safest approach regardless.

Now let's dig in and do some testing. First, I was curious why the compiler didn't pick up on this issue. After all, even raw pointers require a `const` or `mut` keyword, and `ioctl` comes from a third-party library (libc, to be specific), so maybe it is defined incorrectly? A quick check of [docs.rs](https://docs.rs/libc/0.2.66/libc/fn.ioctl.html) reveals the function signature.

```rust
pub unsafe extern "C" fn ioctl(fd: c_int, request: c_ulong, ...) -> c_int
```

Well that answers my first question. The compiler didn't pick up on this issue because `ioctl` only defines its first two arguments explicitely. Any other arguments do not have a defined type, so Rust can't type-check them.

My next question is then, was the original code *really* undefined behavior? It seems likely, but I'd be happier if I *knew* rather than *assumed*. How can we test that? I came up with one way, which admittedly involves a bit of cheating. The only reason the above code *wouldn't* be UB is because the mutation crosses an FFI boundry, so our test must simulate that.

The test code:

```rust
unsafe extern "C" fn put_one(x: *mut u64) {
    *x = 1
}
fn main() {
    let x = 2u64;
    let x_ptr = &x as *const u64 as usize;
    unsafe { put_one(x_ptr as _) }
    println!("x: {}", x);
}
```

([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=246270064e900087a0e969a93b3c05d3))

So here is the "cheat": our FFI call isn't actually calling C code, it's just an FFI wrapper around Rust. The benefit is that we can analyze this code with Miri, a Rust tool that helps detect undefined behavior. Miri works by compiling the original code into an intermediate language (MIR) and interpreting it. Generating MIR is part of the normal compilation process, which is important because it means the compiler could perform optimizations on the generated MIR before compiling it down to machine code. On top of that, the compiler may use some assumptions when optimizing code. Any code that fails to maintain these assumptions could be "optimized" and end up doing just about anything. In other words, such code constitutes undefined behavior.

A quick note: Miri is an excellent tool, however it cannot detect all kinds of UB. See the project [README](https://github.com/rust-lang/miri/blob/master/README.md) for more details.

The line `let x_ptr = ...` exists primarily to try and "trick" the compiler. Normally, type-checking would prevent us from creating a mutable pointer to x, however we want to get around that for this test. In addition, it would be great (for this test) if the compiler would just "forget" that x_ptr is in any way related to x, so we cast it to a `usize` temporarily. At this point, the "pointer" is actually just some integer value.

If you run this code in the Rust playground, at least as of the time of writing, you'll see the code works as you might naively expect. That is to say, the value of `x` as printed is 1. However if you run Miri (Tools -> Miri), you'll see there is a problem. Miri detects `no item granting write access to tag <untagged> found in borrow stack`. What this really means is that we are writing to a memory location that the compiler assumes we won't write to. That seems fair since `x` wasn't declared with `mut`, and it does support the idea that the original code is UB.

But we can dig deeper. My next question is, precisely what does it take to make Miri happy? The first change is to make `x` mutable: `let mut x = 2u64;`. Miri still doesn't like it, which tells us it isn't *just* about whether `x` is mutable, it's also about how x is borrowed. In fact, if you look at the Miri error again, you'll see the phrase "in borrow stack", indicating it is tracking the borrows. If we think about it, even though x is declared as mutable, we only borrow it immutably, so the compiler could still assume its value is never changed. That would be bad.

So let's tell the compiler we are borrowing mutable. `let x_ptr = &mut x as *const u64 as usize;`. Now x is mutable *and* we borrow it mutably. Miri, however, is still unhappy with this code and for the same reason as before (*See [Following Up](#Following-Up)). We'll try `let x_ptr = &mut x as *mut u64 as usize;` now. And finally, Miri is happy! That tells us that even though we borrow x mutably, the act of converting the mutable borrow into an immutable pointer and back is enough to cause UB. Good to know.

Now that Miri is happy, we're done here, right? Well, no. There is still one thing I'd like to look at: `UnsafeCell`. Up until now, we've had UB because we wrote code in a way that allowed the compiler to make bad assumptions about our intentions. Every time the compiler assumes we won't change a value and we do, Miri gets mad and tells us about it. `UnsafeCell` allows us to tell the compiler to stop making assumptions about the contained value. Let's take a look at some code:

```rust
use std::cell::UnsafeCell;

unsafe extern "C" fn put_one(x: *mut u64) {
    *x = 1
}
fn main() {
    let x = UnsafeCell::new(2u64);
    let x_ptr = &x as *const UnsafeCell<u64> as usize;
    unsafe { put_one(x_ptr as _) }
    println!("x: {}", x.into_inner());
}
```

([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=7e9e1d524054c3ff42e99dab7526ac7d))

Here, we again take an immutable borrow of x, and mutate the value behind it. As with the original test, x is not even declared mutable. And yet, Miri has no issue with this code. Per [documentation](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html), "UnsafeCell<T> is the only core language feature to work around the restriction that &T may not be mutated." So we were able to work around the compilers assumptions that were giving us errors before.

`UnsafeCell` does not disable *all* of the compilers assumptions around borrows and mutability, however. Miri is unhappy with this code, for instance:

```rust
use std::cell::UnsafeCell;

unsafe extern "C" fn put_one(x: *mut u64) {
    *x = 1
}
fn main() {
    let x = UnsafeCell::new(2u64);
    let x_ptr = &x as *const UnsafeCell<u64> as usize;
    let x_mut_ref1: &mut u64 = unsafe { &mut *(x_ptr as *mut u64) };
    let x_mut_ref2: &mut u64 = unsafe { &mut *(x_ptr as *mut u64) };
    unsafe { put_one(x_mut_ref2) }
    println!("x: {}", x_mut_ref1);
}
```

([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=d0e35d4877c36956f1c58fc1bda0a549))

Here, we break a different rule in Rust: no two mutable borrows may reference the same memory. `UnsafeCell` does not change this assumption, and so even when using `UnsafeCell`, we still need to be wary of creating mutable references to it.

So to summarize what we've seen: if you intend to mutate something, even across FFI boundaries, always delcare it mutable, always take a mutable reference, and never convert said reference directly to an immutable pointer. The only exception being if you use `UnsafeCell` to disable Rust's normal assumptions.

## Following Up

It's come to my attention that in some instances it might be okay to convert a `*const` to a `*mut` and mutate it. However the conversion from `&mut` to `*const` is not as simple as it seems. Let's take a closer look:

```rust
unsafe extern "C" fn put_one(x: *mut u64) {
    *x = 1
}
fn main() {
    let mut x = 2u64;
    let x_ptr: &mut u64 = &mut x;
    let x_ptr: *const u64 = x_ptr;
    unsafe { put_one(x_ptr as _) }
    println!("x: {}", x);
}
```

([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=38986879a9d2340ddcf984eef4f75aad))

This is very similar to code seen earlier. A couple differences are I'm being more explicit in pointer conversions, and skipping `usize` conversions. The conversion we do here is equvilant to `let x_ptr = &mut x as *const u64;` from before. As already established, it fails Miri. If we change `x_ptr: *const u64` to `x_ptr: *mut u64` in the code above, Miri is happy again. And it is at *this point* it is okay to convert the pointer to a `*const`. This code is completely fine according to Miri:

```rust
unsafe extern "C" fn put_one(x: *mut u64) {
    *x = 1
}
fn main() {
    let mut x = 2u64;
    let x_ptr: &mut u64 = &mut x;
    let x_ptr: *mut u64 = x_ptr;
    let x_ptr: *const u64 = x_ptr;
    unsafe { put_one(x_ptr as _) }
    println!("x: {}", x);
}
```

([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=699539ef06f1efcd6b95827174f571d1))

We can see this is in Rust's standard library as well. The definition of [`NonNull`](https://doc.rust-lang.org/src/core/ptr/non_null.rs.html) is simple:

```rust
pub struct NonNull<T: ?Sized> {
    pointer: *const T,
}
```

`NonNull` also includes this conversion:

```rust
impl<T: ?Sized> From<&mut T> for NonNull<T> {
    #[inline]
    fn from(reference: &mut T) -> Self {
        unsafe { NonNull { pointer: reference as *mut T } }
    }
}
```

The conversion `reference as *mut T` is not strictly necessary for the code to compile. However without that conversion, Miri would be unhappy anytime we mutated the underlying `T`.

On a final note, many of these concepts are related to an aliasing model called [Stacked Borrows](https://www.ralfj.de/blog/2019/05/21/stacked-borrows-2.1.html), which is the model Miri is based on. Rust, however, does not yet have an official aliasing model, so precisely what constitutes UB is subject to change. A detailed paper on the model can be found [here](https://plv.mpi-sws.org/rustbelt/stacked-borrows/paper.pdf). I won't claim to fully understand this model, however as I explore and test the ideas of this model, I may make additional posts.
