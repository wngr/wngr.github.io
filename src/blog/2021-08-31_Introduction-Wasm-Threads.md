# Basic Introduction to WebAssembly Threads in Rust

Quite some introductions have been written how to get started compiling Rust to
WebAssembly and using it in the browser. The inertia behind wasm threads however
fell prey to the Spectre/Meltdown mitigation measures introduced in early
2018[^2]. Fortunately, in the last 1.5 years the bits and pieces that make up
wasm threads were re-added to both Firefox and Chromium.

Wasm threads are not a atomic thing like POSIX threads, but rather are made up
of composable pieces. The main idea is simple really: Spawn wasm code into
mulitple web workers, and give them a shared slice of memory -- what could
possibly go wrong? Anyway, in order to synchronize access to the
`SharedArrayBuffer`[^3], a set of atomic instructions[^4] was introduced.
Those features are available for "cross origin isolated" websites, meaning that
a set of custom headers must be provided while serving the top level document,
and cross origin inclusions are restricted[^5].

Let's take it step-by-step, from a wasm module loaded into the main js context,
over a wasm module loaded in a web worker, to a wasm module loaded in multiple
web workers and them communicating via shared memory.

Loading a wasm module on the browser's main thread is quite simple[^6] with the
right incantations of `wasm-bindgen`:
```rust,ignore,noplayground,ignore
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {}!", name));
}
```
and is loaded inside `index.html` as a ES module:
```html
<html>
  <head>
    <meta content="text/html;charset=utf-8" http-equiv="Content-Type"/>
  </head>
  <body>
    <script type="module">
      import init, { greet } from './pkg/wasm_hello_world.js';

      async function run() {
        await init();

        greet("World");
      }
      run();
    </script>
  </body>
<html>
```

Spawning the wasm module inside a web worker does not require any changes to the
wasm module itself, just some glue code to spin up the `worker.js`[^7]:
```js
import init, { greet } from './pkg/wasm_hello_world.js';

self.onmessage = async event => {
  await init();
  greet(event.data);
}
```
and the `index.html` is adapted to:
```html
    <script type = "module">
      let worker = new Worker("./worker.js", { type: "module" });
      worker.postMessage("World!");
    </script>
```

Now that we put that behind us, it get's more interesting with multiple
workers[^8]. The basic workflow is:
1. Spawn workers.
2. Initialize wasm module with a shared memory slice (`SharedArrayBuffer`)
   inside each worker.
3. Run your application.

The wasm module gets a custom setup routine, where the workers are spawned, so
we don't have to write javascript (:shudder:).
```rust,ignore,noplayground,ignore
#[wasm_bindgen]
pub fn setup(n: usize) -> Promise {
    console_error_panic_hook::set_once();

    let mut opts = WorkerOptions::new();
    opts.type_(WorkerType::Module);
    for i in 0..n {
        log(&*format!("Starting worker {}", i));
        opts.name(&*i.to_string());
        let worker = Worker::new_with_options("./worker.js", &opts).unwrap();
        let arr = js_sys::Array::new();
        arr.push(&wasm_bindgen::module());
        arr.push(&wasm_bindgen::memory());

        worker.post_message(&arr).unwrap();
    }
}
```
The web worker is created, and a js array with the "wasm module" and the shared
memory slice is sent to it. There are only a handful of objects that are passed
through from the browser via the `postMessage` interface incl. those two. The
invoked `worker.js` code in the web worker looks like:
```js
// First message is init
self.onmessage = async event => {
  let [module, memory] = event.data;

  let { default: init, entry } = await import('./pkg/wasm_hello_world.js')
  await init(module, memory);
  entry();

  // We don't expect any further messages
  self.onmessage = () => {
    throw new Error("Unexpected");
  }
}
```
The worker is initializing the wasm module with the call to the modules' default
export, and then invoking the `entry` founction, which again is defined in the
Rust code:
```rust,ignore
#[wasm_bindgen]
pub fn entry() {
    let name = js_sys::global()
        .unchecked_into::<DedicatedWorkerGlobalScope>()
        .name();
    log!("Hello from Worker {}", name);
}
```
And that code is run within each worker's thread -- mission accomplished! You
can find the full code incl. build steps of the three examples above in the
linked repository.

So this is pretty basic, but it gets fairly complicated when trying to integrate
that approach into larger js application; let alone trying to bundle it together
with Webpack etc. In my next post, I'm going to introduce a library hiding all
the dirty details behind a (hopefully) easy to use API.

--------

[^2]: [https://blog.mozilla.org/security/2018/01/03/mitigations-landing-new-class-timing-attack/](https://blog.mozilla.org/security/2018/01/03/mitigations-landing-new-class-timing-attack/])

[^3]: [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)

[^4]: [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)

[^5]: [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#security_requirements](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#security_requirements)

[^6]: [https://github.com/wngr/wasm-hello-world/tree/master/main-context](https://github.com/wngr/wasm-hello-world/tree/master/main-context)

[^7]: [https://github.com/wngr/wasm-hello-world/tree/master/single-web-worker](https://github.com/wngr/wasm-hello-world/tree/master/single-web-worker)

[^8]: [https://github.com/wngr/wasm-hello-world/tree/master/multi-web-worker](https://github.com/wngr/wasm-hello-world/tree/master/multi-web-worker)


