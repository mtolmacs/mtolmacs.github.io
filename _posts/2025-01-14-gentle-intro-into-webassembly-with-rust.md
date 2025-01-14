---
layout: post
title: A Gentle Introduction to WebAssembly in Rust (2025 Edition)
---

It’s clear [WebAssembly](https://webassembly.org/) is one of the more popular up-and-coming technologies out there. Its promise, a universal executable format, is not new. In fact it dates back to 1995 (almost thirty years ago!) with Java. Arguably, Java was successful in some areas, many enterprise software is built on Java after all, it tried for a brief time (Java Web Start) and eventually failed to ride the stellar rise of the world wide web. Microsoft .NET is a younger contender, but it arguably suffering from the same adoption challenge as Java. While it can run on most systems now, the web is still not one of them.

Enter WebAssembly (or WASM for short), supported by a wide consortium of players, developed in the open and as an open standard, with the WEB as its primary platform. While it’s too early to tell if WebAssembly will be the winner we’ve been waiting for, its adoption is wide enough, the core technology is stable enough that it’s worth considering it for even professional cases. If in doubt, just consider that [Figma](https://www.figma.com), the interface design software, is [built on C++ and WebAssembly](https://www.figma.com/blog/webassembly-cut-figmas-load-time-by-3x/).

Why is a portable, widely supported executable format is such a big deal, you ask? One of the main reasons is that there are a LOT of software already written and most of them are complex systems, not easily ported to other languages and tooling. 99% of the time these software is written in C or C++. WebAssembly offers direct compilation from C, C++ and many more languages an environment (including Rust!) without major hiccups. And that is a previously unseen capital Bid Deal! Besides making software porting almost trivial it’s also a nice benefit that it can often run compute intensive tasks faster than JavaScript.

So in this guide we’ll walk through setting up the tooling and development environment for building and using WebAssembly in Rust, embedding it in a TypeScript project, review how communication between TypeScript and Rust can happen, then finally how you can debug your WebAssembly directly in the browser and/or your favorite IDE. I will use Visual Studio Code as the IDE and Chrome as the browser, but apart from some debugging options, you can reproduce these in your tool of choice.

> _Note: This guide heavily relies on the excellent_ [_Rust WASM Book_](https://rustwasm.github.io/docs/book)_, which contain a lot more examples and details than this article. I recommend checking it out after finishing this one._

## What exactly is WebAssembly?

As mentioned previously, WebAssembly is an open standard of a binary 32 bit instruction set architecture ([ISA](https://en.wikipedia.org/wiki/Instruction_set_architecture)) for a stack-based, sandboxed virtual machine. That’s a heavy load of [_terminus technicus_](https://www.oxfordreference.com/display/10.1093/acref/9780195369380.001.0001/acref-9780195369380-e-1999), but it’s an apt summary.

In plain English, it means that,

- Unlike JavaScript, its final form is binary, not text, similar to how programs on your machine are in binary format.
- It is a virtual machine by virtue of not running directly on your hardware as its binary code is translated on-the-fly when you run it, offering the portability you’d expect from a web technology.
- Sandboxed, so any interaction with the outside world is carefully scrutinized and approved by the end user, which offers safety guarantees for users that when they load a WebAssembly module on a web page, it cannot access their data or modify their system, unless explicit permission is given.
- It’s 32 bit, meaning that we can allocate a maximum of 4Gb of RAM for our WebAssembly application (until [WASM64](https://github.com/WebAssembly/memory64) comes around).
- Stack-based is… you know what?! Let this be a concern for compiler developers and let’s get to coding!

## Setting Up Tooling

The reference implementations and the most mature WebAssembly development pipeline called [Bynarien](https://github.com/WebAssembly/binaryen) is still built around C/C++, mainly because the amount of useful code people want to run in the browser was built with C/C++. The Rust community is building it’s own WebAssembly pipeline, however it’s in a state of Tier 2 without Host Tooling at the beginning of 2025. This means that while it is easily and safely used by developers even for production purposes, it lacks some native tooling. This is where we will rely on the Bynarien toolbox to patch in the holes where the Rust WASM pipeline is lacking.

Let’s install the required tools and set up the project:

1. Install yarn (or npm, or pnpm, if you don’t have it already)

```bash
> corepack enable    # I'll use corepack because I have node@20
```

2. Start a new TypeScript project called ‘wasm-on-web’ with [Vite](https://vite.dev/) (or your framework of choice, if any)

```bash
> yarn create vite wasm-on-web --template vanilla-ts
```

3. Commit the current state to Git because we will overwrite some files and you’ll need the old ones

```bash
> git init   # If you haven't initialized the git repo yet
> git commit -m "Initial setup"
```

3. Install Rustup to manage your Rust installation and toolchains

```bash
> curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

4. Set the Rust channel to stable

```bash
> rustup default stable
```

5. Download the compilation toolchain for WASM

```bash
> rustup target add wasm32-unknown-unknown
```

6. Install some tools we’ll use during our exercise

```bash
> cargo install wasm-tools
> cargo install wasm-opt
> cargo install wasm-pack
```

7. Install the Bynarien toolkit for our investigations. I use brew, but on Windows you might need to [build it yourself](https://github.com/WebAssembly/wabt?tab=readme-ov-file#building-windows) or use a package manager like Scoop and use the [pre-built package from extras](https://scoop.sh/#/apps?q=wabt).

```bash
> brew install wabt
```

8. Install `cargo-generate` to quickly scaffold our Rust project over the Vite project we created in step 2

```bash
> cargo binstall cargo-generate   # Install pre-built binary
```

> Note: cargo-generate needs libssl-dev (openssl) installed if you use `cargo install cargo-generate`

8. Overlay the Rust project of our TypeScript / Vite project

```bash
> cd wasm-on-web
> yarn install
> cargo generate              \
        --init                \
        --name wasm-on-web    \
        --overwrite           \
        --git https://github.com/rustwasm/wasm-pack-template
```

> Note: `cargo-generate` needs to overwrite some files, because it conflicts with Vite, but the only thing you need to merge is `.gitignore`. You’ll need both the original lines and the newly added ones.

9. Test if everything works so far

```bash
> cargo test
> yarn dev  # Should open a web browser
```

With `Ctrl + C` you can exit the Vite server. You can also commit it into Git now.

## Sidenote: Publish your Rust WASM package on [npmjs.com](http://npmjs.com)

If your WASM code is self contained in Rust, you can build it in production mode and publish it on [npmjs.com](http://npmjs.com) right now. The `wasm-pack` tool creates all the TypeScript types, package.json skeleton and anything else needed for a complete package. It is recommended that you review and update your `package.json` file prior to publishing.

```bash
> wasm-pack build
> yarn publish
```

## Build and Integrate the WebAssembly Module

1. We need to build the WASM module so we can import it in the TypeScript project

```bash
> wasm-pack build --dev
```

2. Add the WASM package to our TypeScript project

```bash
> yarn add link:./pkg
```

> Note: It is important to add our WASM package as ‘link’, otherwise when we rebuild the WASM module Vite will not pick up the new version!

3. Add the Vite plugins required to interoperate with WASM

```bash
> yarn add -D \
    vite-plugin-wasm \
    vite-plugin-top-level-await \
    vite-plugin-wasm-pack-watcher
```

4. Configure the WASM loader in Vite by creating `vite.config.dev.ts` and adding the following contents:

```javascript
import { defineConfig } from "vite";
import wasm from "vite-plugin-wasm";
import topLevelAwait from "vite-plugin-top-level-await";
import wasmPackWatchPlugin from "vite-plugin-wasm-pack-watcher";

export default defineConfig({
  build: {
    watch: {
      include: ["src/**/*.ts", "src/**/*.rs"],
    },
  },
  plugins: [wasmPackWatchPlugin(), wasm(), topLevelAwait()],
});
```

5. Create the production Vite configuration file `vite.config.ts` and add the following contents:

```javascript
import { defineConfig } from "vite";
import wasm from "vite-plugin-wasm";
import topLevelAwait from "vite-plugin-top-level-await";

export default defineConfig({
  plugins: [wasm(), topLevelAwait()],
});
```

> Note: We can’t use the same config file because of the watch configuration, it would hang the `vite build` command when we build for production.

6. Add the `npm-run-all` package as a dev dependency in preparation for the next step below:

```bash
> yarn add -D npm-run-all
```

> Note: This is a platform-independent way to run npm scripts one after the other with the `run-s` shortcut, which makes it possible for the Windows folks to follow this tutorial without issues. Even if you don’t use Windows it’s polite to have a solution in place which works for all.

7. Modify “_scripts_” section in `package.json` to call `wasm-pack` before starting Vite for all configurations, so a fresh WASM build is always ready for us at the start dev start and the final WASM module is build in release mode before vite production build happens:

```json
{
  ...
  "scripts": {
    "build": "run-s wasm-pack:release tsc vite:build",
    "dev": "run-s wasm-pack:dev vite:dev",
    "tsc": "tsc",
    "vite:build": "vite build",
    "vite:dev": "vite -c vite.config.dev.ts",
    "vite:preview": "vite preview",
    "wasm-pack:dev": "wasm-pack build --dev",
    "wasm-pack:release": "wasm-pack build"
  },
  ...
}
```

8. Start the Vite dev mode and continue with writing our Rust WASM code

```bash
> yarn dev
```

## Exporting and Importing in Rust

Opening up `src/lib.rs` we can see that we have a `greet()` function already exported and ready for us to be called. We know it’s exported because it has the `[#wasm_bindgen]` macro applied to it from the wasm-bindgen package.

```rust
#[wasm_bindgen]
pub fn greet() {
    alert("Hello, wasm-on-web!");
}
```

> Note: You can in fact add the `#[wasm_bindgen]` macro to enums, structs and impls, not just stand-alone fns!

There’s another trick in this file above the `greet()` function, which you’ll use when you want to call JavaScript functions:

```rust
#[wasm_bindgen]
extern "C" {
    fn alert(s: &str);
}
```

This specifices an `extern` block which contain external (JS) functions we want to call from our Rust WASM module. These can be called with the C calling convention, not the Rust calling convention, hence `extern “C”`. These should be wired up between JS and Rust too, so you need to add the `#[wasm_bindgen]` macro here as well. Not unsurprisingly, this is **unsafe** and since the Rust compiler won’t be able to verify whether the external function exists, it’s calling signature (parameters) are properly typed, present and ordered the right way. Specifying these incorrectly very likely will crash your program.

## Importing From and Exporting To WASM in TypeScript

Now it’s time to open `src/main.ts` and import our new Rust WASM module at the top of the file.

```javascript
import { greet } from "wasm-on-web";
```

Then on the bottom, just simply call the `greet()` method we just imported.

```javascript
// Call the greet function from WASM with the
greet();
```

All the translation, loading the WASM module and configuration is being taken care of by `wasm-bindgen`, the Rust package our template installed. Even TypeScript type definitions are generated for us.

If everything went well and `yarn dev` is running, the browser is open, we’ll see the alert right from the Rust code.

![](public/assets/posts/2025-01-14-gentle-intro-into-webassembly-with-rust/hello-wasm-on-web.png)

If you want to call your JavaScript functions in Rust, you already saw how it is done with the `alert()` JS function. One additional step is required, namely that you have to add your function to the global `window` object.

```javascript
// Just to resolve TypeScript errors
declare global {
  interface Window {
    jsFunction: () => void;
  }
}

window["jsFunction"] = () => {
  alert("Hello from JS!");
};
```

Then you’re ready to declare and call it in Rust.

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    fn alert(s: &str);
    fn jsFunction();
}

#[wasm_bindgen]
pub fn greet() {
    jsFunction();
}
```

## Passing Primitive Parameters Back and Forth

A function call seldom worth much without passing data to it in the form of parameters. `wasm-bindgen` takes care of this too in simple cases and almost completely in heap allocated types. For example, if you want to pass a custom name to greet to our greet function implemented in Rust, you could do this:

```rust
#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(format!("Hello {}!", name).as_str());
}
```

On the TypeScript side, just simply pass the parameter and we’re ready to go:

```rust
greet("this is TS");
```

It should show the alert dialog with our new parameter:

![](public/assets/posts/2025-01-14-gentle-intro-into-webassembly-with-rust/this-is-ts.png)

For the other direction, we’re already seen with the native `alert(…)` function, just pass the string slice to the JS function and `wasm-bindgen` takes care of it.

So what happens when you need to use complex types, maybe heap allocated types as parameters? That’s what we’re dealing with next.

## Using existing JavaScript APIs and Functions

I don’t know about you but I’ve had just about enough of the alert dialog and would like to use `console.log(…)` and similar native JS APIs. We could wire them up manually, figure out the complex parameter definition, but there is an easier way: The `web_sys` package. Let’s install it! Add this to your `Cargo.toml` file:

```toml
[dependencies.web-sys]
version = "0.3"
features = [
  "Window"
]
```

We can get rid of our manually implemented `alert()` mapping and use the `web_sys` `console.log` implementation going forward:

```rust
#[wasm_bindgen]
pub fn greet(person: &str) {
    web_sys::console::log_1(&format!("Hello {}!", person).into());
}
```

## Passing Complex Parameters Back and Forth

With complex types we need to consider the fact that what’s a complex type in one language might not be so in the other. A typical example of this is String. String is an owned, complex type in Rust and behaves like a primitive in JavaScript. `wasm-bind` hides this difference by making a copy in WASM memory of your JS string when you expect a `String` parameter type in Rust. You can make it mutable, but it will only modify the Rust copy of the `String`, it will not propagate back to the JS string. In order to have two-way communication, we have to put in some legwork.

If you have a JS class or object which you’d like to receive as a parameter on the Rust side, you’ll have to define the mapping as an `extern “C”` block. Let’s say we have a JS class defined in TS:

```javascript
class TSDef {
  constructor(public id: string) {}

  run() {
    console.log(this.id);
  }
}
```

On the Rust side if we want to access the `id` property and the `run` method, we have to map it and use it the following way:

```rust
#[wasm_bindgen]
extern "C" {
    pub type TSDef;

    // Uses the JS "get()" method which is
    // provided by the "class" base prototype chain
    #[wasm_bindgen(method, getter)]
    fn id(this: &TSDef) -> String;

    // Uses the JS "set()" method which is
    // provided by the "class" base prototype chain
    // NOTE: "set_<property>" naming is important!
    #[wasm_bindgen(method, setter)]
    fn set_id(this: &TSDef, val: &str);

    #[wasm_bindgen(method)]
    fn run(this: &TSDef);
}

#[wasm_bindgen]
pub fn remote_instance_param(tsdef: &TSDef) {
    // Display the id of the instance
    alert(tsdef.id().as_str());

    // Modify the id on the JS instance
    tsdef.set_id("zyxw");

    // Call a method on the JS instance
    tsdef.run();
}
```

Now if you call the `remote_instance_param()` function from TypeScript, you’ll see an alert with the original “abcd” message from JS and a console message with “zywx” from the `run()` method invoked from Rust reading the modified `id` value and printing it on the console.

What if we want to expose a Rust type to JS? A similar mapping needs to take place. Let’s have a `Person` struct which we would like to use in JS:

```rust
#[wasm_bindgen]
pub struct Person {
    pub id: u32,
    name: String, // String is not Copy, so we cannot make it public!
}

// Implement a constructor and a getter/setter for the String field
#[wasm_bindgen]
impl Person {
    #[wasm_bindgen(constructor)]
    pub fn new(id: u32, name: String) -> Person {
        Person { id, name }
    }

    // Getter which automatically gets called on the JS side
    #[wasm_bindgen(getter)]
    pub fn name(&self) -> String {
        self.name.clone()
    }

    // Setter which automatically gets called on the JS side
    #[wasm_bindgen(setter)]
    pub fn set_name(&mut self, name: String) {
        self.name = name;
    }
}
```

In the TS/JS side, we can then simply import it and use it with one big caveat:

```javascript
import { greet, Person } from "wasm-on-web";

// Create a Person object which is shared between WASM and JS
const person = new Person(2343, "John");

// Automatically call the setter on the Rust side
person.name = "Jane";

// Call the greet function from WASM with the
greet(person);

// Don't forget to free the WASM memory when you're done with the shared object!
person.free();
```

Because the `Person` type is a WASM type defined in Rust, we need to take care of the de-allocation ourselves when we no longer need it. Unfortunately there is no `Drop` mechanic and the memory of JS-allocated Rust native types need to be freed manually!

## Debugging WebAssembly

Now that we have seen how we can transfer data between the two sides, we also became aware of how fragile is the whole setup. Having a robust debugging workflow is essential to quickly identify and root out the inevitable issues.

To date the best option to do so is via a Google Chrome extension, [C/C++ DevTools Support (DWARF)](https://chromewebstore.google.com/detail/cc++-devtools-support-dwa/pdcpmagijalfljmkmjngeonclgbbannb), built by Google’s Engineers. DWARF is one of the debugging information formats for binary code and it’s the type of debug info Rust generates by default, so it’s ideal for our debugging needs. Unfortunately this is Chrome only, so you’ll have to stick with Chrome.

Once you added this extension to your Chrome, restart the browser so it loads the extension proper. Next you’ll need to enable a setting in the DevTools settings panel (see the cog icon on your DevTools panel after you opened it). It’s called “_Allow DevTools to load resources, such as source maps, from remote file paths. Disabled by default for security reasons._”

![](public/assets/posts/2025-01-14-gentle-intro-into-webassembly-with-rust/chrome-preferences.png)

Now the browser is ready, we need to set up the Rust side. First, make sure you have the Rust source code installed, because the stack traces most definitely will go into the Rust standard library and having a nice source view into those files will help us figuring out what’s going on much easier than without it.

```bash
> rustup component add rust-src
```

We also need to modify our `Cargo.toml` because by default `wasm-pack` strips out the DWARF debug information. Add this to the end of `Cargo.toml`:

```toml
[package.metadata.wasm-pack.profile.dev.wasm-bindgen]
# Keep the DWARF debug info for debugging in dev mode
dwarf-debug-info = true
```

> Note: You may need to clear your browser cache in order for this to work.

Now if you start your Vite project with `vite dev` and load it up in Chrome, then check out the DevTools console, you should see something like this:

![](public/assets/posts/2025-01-14-gentle-intro-into-webassembly-with-rust/dwarf-extension.png)

This is a good sign that the Chrome extension found the debug information and loaded it into the DevTools.

> Note: If you can’t see this message and can’t debug your WASM, then check the generated \*.wasm files for DWARF debug info. Install `wasm-objdump` from HomeBrew and see if you can find “.debug\_str”, “.debug\_line” and similar custom sections in your WASM. If not, then the DWARF debug info is missing, therefore the extension has nothing to work with.
> 
> You can also run your project with `RUST_LOG=info yarn dev` to see if the `wasm-pack` step runs `cargo build` with the `—keep-debug` parameter.
> 
> Finally if you run `yarn dev —debug` you can see what files the browser requests from Vite and if you see it requesting `*.dwg` file(s), then the browser extension is working properly, your WASM file should be the problem.

If you open up your sources tab in DevTools, you’ll see that there is a `file:///` major section previously missing. Opening it you’ll find your project directory, in it you’ll see the Rust source file, `lib.rs` loaded and available. You can now set breakpoints in it and debug like you would a JS file. You can also see the variable values, which might not be immediately useful, but often enough to figure out what’s going on.

One other benefit this extension brings us is if there is a panic in our code, we will see the stack trace in console with rust file names, line numbers and character positions! Certainly helpful!

![](public/assets/posts/2025-01-14-gentle-intro-into-webassembly-with-rust/resolved-exception.png)

> Note: If you see an error message on the Sources tab that refers to files cannot be open, specifically when it starts with `file:///rustc/<hash>` then you need to set up a mapping to your rust sources.

In case you need to map some directories from the WASM debug informatin, maybe because you built it on a remote machine and your files are in a different folder, there is a way to do it in the Chrome extension settings. Open it up and add the directory mapping from the `/rustc/<hash>/` to the absolute path of your Rust sources.

![](public/assets/posts/2025-01-14-gentle-intro-into-webassembly-with-rust/extension-settings.png)

My mapping was `/rustc/90b35a6239c3d8bdabc530a6a0816f7ff89a0aaf` => `/home/mtolmacs/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust`. Yours might be different.

In case you need to figure out the Rust toolchain directory on your system, you can use the following command:

```bash
> rustc --print sysroot
```

## Sidenote: WebAssembly on the server-side

While eminently useful for bringing large-scale applications like Figma to the web, it’s gaining popularity on the server-side as well. Highly distributed apps use it as a platform agnostic edge runtime and the Web3 community found use for it as an open platform for smart contract runtimes on the blockchain. One other interesting use-case is [platform-independent plugins](https://swc.rs/docs/plugin/publishing) for tools like the SWC JavaScript / TypeScript transpiler created by Vercel.

The solution all these projects use is called WASI, the WebAssembly System Interface, which is a true standard library to access all system resources, just like you would with Rust or C. However WASI is quite young and still not a stable standard. [NodeJS just started to support running WebAssembly with WASI in node@23](https://nodejs.org/api/wasi.html), so it is quite experimental yet. Other runtimes like `wasmtime` can run the WASI Preview 1 and is reasonably stable.

So as a final act, let’s download and build a sample WASM app using WASI to open stdio handlers and uses SWC to transpile the input, the output the result.

```bash
> git clone https://github.com/zebp/wasi-example-swc
```

Time to add the Rust WASI runtime.

```bash
> rustup target add wasm32-wasip1
```

Time to compile it to WASM WASI.

```bash
> cargo build --target wasm32-wasip1
```

Check if NodeJS is at least 23.

```bash
> node -v
```

Finally create the loader JavaScript which loads the WASM (change the `preopens` mapping to avoid any errors)

```javascript
'use strict';
const { readFile } = require('node:fs/promises');
const { WASI } = require('node:wasi');
const { argv, env } = require('node:process');
const { join } = require('node:path');

const wasi = new WASI({
  version: 'preview1',
  args: argv,
  env,
  preopens: {
    '/': '/Users/mtolmacs/Projects/wasi-example-swc/',
  },
});

(async () => {
  const wasm = await WebAssembly.compile(
    await readFile(join(__dirname, '../target/wasm32-wasip1/debug/swc-wasi.wasm')),
  );
  const instance = await WebAssembly.instantiate(wasm, wasi.getImportObject());

  wasi.start(instance);
})();
```

Time to run it:

```bash
src > cat examples/async-generator.js | node loader.js
```

If you want to use `wasmtime`, just install it and run the WASI WASM directly!

```bash
> brew install wasmtime
```

```bash
> cat examples/async-generator.js | wasmtime target/wasm32-wasip1/debug/swc-wasi.wasm
```

## Continue learning about Rust and WebAssembly

If you’d like to continue learning about Rust and WebAssembly, I highly recommend reading through the [Rust WebAssembly Book](https://rustwasm.github.io/docs/book/) and then follow up with the [The `wasm-bindgen` Guide](https://rustwasm.github.io/wasm-bindgen/introduction.html) for the practical training material.