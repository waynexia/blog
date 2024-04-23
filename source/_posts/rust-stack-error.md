---
title: How error occurs in GreptimeDB
date: 2024-01-22 00:53:48
tags: "Rust"
---

>TL;DR:
>
>This post discusses the practice of Rust error handling topic in GreptimeDB. Including how to build a cheaper yet more accurate error stack to replace system backtrace, how to organize errors in large project and how to print errors in different schemes to log and end users. And shares possibly future work in the end.
>
>An example of error in GreptimeDB looks like:
>```
0: Foo error, at src/common/catalog/src/error.rs:80:10
1: Bar error, at src/common/function/src/error.rs:90:10
2: Root cause, invalid table name, at src/common/catalog/src/error.rs:100:10
>```

# Introduce

## What `Error` is in Rust

<!-- toc:
- What `Error` is in Rust. -->

In Rust, functions that might fail usually return a special enum [`Result<T, E>`](https://doc.rust-lang.org/std/result/enum.Result.html#variant.Err), where `E` usually implements the trait [`std::error::Error`](https://doc.rust-lang.org/std/error/trait.Error.html). This is the fundamental of error handling.

This blog shares our experience on how to organize variant types of `Error` in a relatively complex system like GreptimeDB, which is composed of multiple components with their own `Error` definition. From how an error is defined to how to present an error to log or end-user.

<!-- todo: add a doc here? -->

## Ecosystem at present

<!-- toc:
- Two widely used crates about error: `thiserror` and `anyhow`.
- How those two are used. Define customized error type.
- Introduce `snafu`, and its characteristics. -->

There are some `Error` structs in std that implement `std::error::Error`, like `std::io::Error` or `std::fmt::Error`. But we would usually define one for our project, as either we want to express our own error info or we may face multiple errors at the same time.

Given the `std::error::Error` trait is not complicated, it's easy to implement manually for the customized error type. However you usually won't want to do so. When error variants increase, it might be hard to work with flooding template code. Nowadays, there are some widely used utility crates to help play around with customized error types, like [`thiserror`](https://docs.rs/thiserror/latest/thiserror/) and [`anyhow`](https://docs.rs/anyhow/latest/anyhow/), from the same author. Said that, use `thiserror` for library project and `anyhow` for binary project. This rule suits a major number of cases.

But for projects like GreptimeDB, where we divide the entire workspace into several individual sub-crates, we need to define an error type for each crate while keeping a streamlined combination. Neither `thiserror` nor `anyhow` can achieve this easily.

Here we choose another crate [`snafu`](https://docs.rs/snafu/latest/snafu/) to instrument our error system. It's like a combination of `thiserror` and `anyhow`. `thiserror` provides a convenient macro to define customized error type, with display, source and some context fields. And `anyhow` gives a `Context` trait that can easily transform from one underlying error into another with a new context.

`thiserror` mainly implements the [`std::convert::From`](https://doc.rust-lang.org/std/convert/trait.From.html) trait for your error type, so that you can easily use [`?`](https://doc.rust-lang.org/std/convert/trait.From.html) to propagate the error you receive. Consequently, this also means you cannot define two error variants that come from the same source. Considering you are performing some I/O operations, you won't know whether an error is generated in write or read. This is also an important reason we don't use `thiserror` -- the context is blurred in type.

# Stacking Error

## What we want from error

<!-- toc:
- In the real world, knowing what the error is is not enough. -->

In the real world, only knowing what an error is is far not enough. Imagine we are building a little protocol component in GreptimeDB (or any other project, just for example), It reads messages from the Internet, decodes, performs some operations and then sends it. We may encounter errors from several aspects:

```rust
enum Error{
    ReadSocket(hyper::Error),
    DecodeMessage(serde_json::Error),
    Operation(GreptimeError),
    EncodeMessage(serde_json::Error),
    WriteSocket(hyper::Error),
}
```

When an error occurs, it might look like `DecodeMessage(serde_json: invalid character at 1)`. But this component might have 10 places that decode the message! How can we figure out in which step we see the invalid content?

So despite the error itself telling what has happened, if we want to have a clue on where this error occurs and if we should pay attention to it, we need the error to carry more information. For comparison, here is an example of an error log you might see from GreptimeDB.

```
Failed to handle protocol
0: Failed to handle incoming content, query: blabla, at src/protocol/handler.rs:89:22
1: Failed to reading next message at queue 5 of 10, at src/protocol/loop.rs:254:14
2: Failed to decode `01010001001010001` to ProtocolHeader, at src/protocol/codec.rs:90:14
3: serde_json(invalid character at position 1)
```

A good error is not only about how it's constructed, but more importantly is what will human see from it. We call it **Stack Error**. It should be intuitive and you must have seen a similar format elsewhere like backtrace.

From this log, it's easy to know the entire thing with full context, from the user-facing behavior to the root cause. Plus the exact line and col number of where each error is propagated. And even gain a tiny report of this error: "in the query "blabla", the fifth package's header is corrupted". It's likely to be an invalid user input and we may not need to handle it from the server side.

This example shows the critical information that an error should contain (and contains in GreptimeDB):
- The root cause. Tells what's happening.
- The full context stack. For debugging or figuring out where this occurs.
- What happens from the user's perspective. Decide whether we need to expose this to user.

The first root cause is often clear in many cases, like the `DecodeMessage` example before. Only if the library or function we used implements their error type correctly. But sometimes it isn't quite helpful if we only have the root cause. Here is another [example](https://github.com/delta-incubator/delta-kernel-rs/pull/151) from DataBricks.

In the following sections, we will focus on the next two points and explain how it's achieved. So hopefully you can achieve the same experience as in GreptimeDB.

## System Backtrace

<!-- toc:
- System Backtrace comes to help
- But backtrace has many problems -->

So, now you have the root cause (`DecodeMessage(serde_json: invalid character at 1)`) but it's not clear at which step this error occurs. Is decoding the header or body?

A natural thought is to capture the backtrace. In a causal where `.unwrap()` is the first choice, the backtrace will show up when error occurs (of course this is a bad practice). It will give you a complete call stack along with the line number. Then inspect the source code stack by stack, after skipping lots of unrelated system stacks, runtime stacks and std stacks, you finally get to the code you write.

Nowadays many libraries also provide the ability to capture backtrace on an `Error` is constructed. Regardless of whether the system backtrace can provide what we truly want, it's very costly on either CPU ([#1261](https://github.com/GreptimeTeam/greptimedb/pull/1261)) and memory ([#1273](https://github.com/GreptimeTeam/greptimedb/pull/1273)). Capturing a backtrace will slow down your program as it needs to walk through the call stack and translate the pointer. Then to be able to translate the stack pointer we will need to include a large `debuginfo` in our binary. In GreptimeDB this means increasing the binary size by > 700MB (4x size compared to 170MB without debuginfo). And there will be many noises in the captured system backtrace because the system can't distinguish whether the code comes from the standard library, a third-party async runtime or our codebase.

There is another difference between the system backtrace and the proposed stack error. System backtrace tells us how to get to the position where the error occurs. While the stack error shows how the error is propagated. Sometimes there are different things. This difference comes from how the two things are implemented.

## Virtual User Stack

<!-- toc:
- Propose the virtual user stack, how does it look like
- The principle, and how it's added to errors -->

Now let's introduce the virtual user stack. The word "virtual" means the contrast of the system stack. Means it's defined and constructed fully on user code. Look closer into the previous example:

```
1: Failed to reading next message at queue 5 of 10, at src/protocol/loop.rs:254:14
```

A stack layer is composed of 3 parts. The first number represents its position in the entire stack. Then follows the description of this layer, which is the [`std::fmt::Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html) implementation of that error. The last is corresponding code location. Rust provides [`file!`](https://doc.rust-lang.org/std/macro.file.html), [`line!`](https://doc.rust-lang.org/std/macro.line.html) and [`column!`](https://doc.rust-lang.org/std/macro.column.html) macros to help get that information.

In practice, we utilize [`snafu::Location`](https://docs.rs/snafu/0.8.2/snafu/struct.Location.html) to gather the code location. So each location points to where the error is constructed. Through this chain we know how this error is generated and propagated to the uppermost.

So here is what it looks like all together:

```rust
#[derive(Snafu)]
pub enum Error {
    #[snafu(display("General catalog error: "))] // <-- the `Display` impl deriv
    Catalog {
        location: Location, // <-- the `location`
        source: catalog::error::Error, // <-- inner cause
    }
}
```

Then we implemented a proc-macro [`stack_trace_debug`](https://greptimedb.rs/common_macro/attr.stack_trace_debug.html) to scrape necessary information from `Error`'s definition and generate the implementation of the related trait [`StackError`](https://greptimedb.rs/common_error/ext/trait.StackError.html), which provides useful methods to access and print the error:

```rust
pub trait StackError: std::error::Error {
    fn debug_fmt(&self, layer: usize, buf: &mut Vec<String>);

    fn next(&self) -> Option<&dyn StackError>;

    // Provided method
    fn last(&self) -> &dyn StackError
    where Self: Sized;
}
```

By the way, we have added `Location` and `display` to all errors in GreptimeDB. This is the hard work behind the methodology.

## Macro Details

Error is a singly linked list, like an onion from outer to inner. So we can capture an error at the outermost and walk through it.

One tricky thing we did here is about how to distinguish internal and external errors. Internal errors all implement the same trait [`ErrorExt`](https://greptimedb.rs/common_error/ext/trait.ErrorExt.html) which can be used as a marker. But depending on this requires a `downcast` every time. We avoid this extra `downcast` call by simply giving a different name to them and detect in our macro. As shown below, we name all external errors `error` and all internal errors `source`. Then return `None` on implementing [`StackError::next`](https://greptimedb.rs/common_error/ext/trait.StackError.html#tymethod.next) method if we find an `error`, or `Some(source)` if we read `source`.

```rust
#[derive(Snafu)]
#[stack_trace_debug]
pub enum Error {
    #[snafu(display("Failed to deserialize value"))]
    ValueDeserialize {
        #[snafu(source)]
        error: serde_json::error::Error, // external source
        location: Location,
    },

    #[snafu(display("Table engine not found: {}", engine_name))]
    TableEngineNotFound {
        engine_name: String,
        location: Location,
        source: table::error::Error, // internal source
    }
}
```

The method [`StackError::debug_fmt`](https://greptimedb.rs/common_error/ext/trait.StackError.html#tymethod.debug_fmt) is used to render the error stack. It would be called recursively in the generated code. Each layer of error will write its own debug message to the mutable `buf`. The content will contain error description captured from `#[snafu(display)]` attribute, the variant arm type like `TableEngineNotFound` and the location from the enumeration.

Given we already defined our error types in that way, adopting stack error doesn't require too much work, only adding the attribute macro `#[stack_trace_debug]` to every error type would be enough.

## Present Error to End User

So far we have done the most things. Here is the last piece about how to present the error to your user.

<!-- toc:
- How error shows to users
- Distinguishing different errors, walk through the error chain -->

Unlike the developer of your system, a user may not care about the line number and even the stack. But what information is helpful to end users?

This topic is very subjective. Our experience is that the leaf (or the innermost) error's message might be useful as it is closer to what really goes wrong. The message can be further divided into two parts: internal and external, where the internal error is those defined in our codebase and the external is from dependencies, like `serde_json` from the previous example. And the root (or the outermost) error's category is more accurate as it comes from where the error is thrown to the user. This can be achieved easily with previous [`StackError::next`](https://greptimedb.rs/common_error/ext/trait.StackError.html#tymethod.next) and [`StackError::last`](https://greptimedb.rs/common_error/ext/trait.StackError.html#method.last). Or you can customize the format you want with those methods.

Combine them, here is the schema of our error that the user would see eventually:

```
KIND - REASON ([EXTERNAL CAUSE])
```

For example:

```
Unexpected Message - Failed to decode `01010001001010001` to ProtocolHeader (serde_json(invalid character at position 1))
```

<!-- But our error here has carried many extra information than the standard library API defined: the source error, the location and the display. Moreover, we have to distinguish internal error (defined in GreptimeDB) and external error (defined elsewhere). -->

## Cost?

The virtual stack is sweety so far. And is cheaper and more accurate than the system backtrace. How much does it cost? 

As for runtime overhead, it only requires some string format for the per-level reason and location.

And it's even better on binary size. In GreptimeDB's binary, the debug symbols occupied ~ 700MB, as a comparison the `strip`-ed binary size is around 170MB, with `.rodata` section size `016a2225` (~ 22.6M), the `.text` section occupies `06ad7511` (~ 106.8M).

Removing all `Location` reduces the `.rodata` size to `0169b225` and the overall binary size to 170MB. while removing all `#[snafu(display)]` reduces the `.rodata` size to `01690225` (~ 22.5M) and the overall binary size to 170MB. Hence this stack error mechanism's overhead to binary size is very low (~ 100K).

# Conclusion and Future Work

<!-- toc:
- In the last, the test about binary size.
- We may consider making it more generic using the new std API -->

In this post, we present how to implement a proc-macro [`stack_trace_debug`](https://greptimedb.rs/common_macro/attr.stack_trace_debug.html) and use it to assemble a low-overhead and powerful stack error message. It also provides a convenient way to walk through the error chain, to help render the error in different schema for different purposes.

This macro is only adopted in GreptimeDB now, we are attempting to make it more generic for different projects. A wide adoption of this pattern can also make it even more powerful by bringing more third-party stacks and detailed reasons.

An unstable API [`provide`](https://doc.rust-lang.org/std/error/trait.Error.html#method.provide) in std `Error` allows getting a field in a struct. It's an option we can consider for refactoring our stack-trace utils.
