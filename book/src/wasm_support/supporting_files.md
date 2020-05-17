# Supporting Files

These steps adds the necessary files support WASM compilation. In brief, these are:

* `index.html`: Web page to display the application.
* `build_release.sh`: Builds the WASM application in release mode.
* `worker.js`: Allows `rayon`'s thread pool to use browser `Worker`s.
* `js/cpal.js`: Allows the WASM application to play audio.
* `js/web_socket_send.js` (optional): Allows the WASM application to send web socket messages.

All of the files are relative to the project root.

<details open>

<summary>1. Create <code>index.html</code>.</summary>

<div style="padding-left: 40px;">

This is the web page that will display the application.

***Important:*** Replace `myapp` with your project's crate name.

```html
<html>
  <head>
    <meta content="text/html;charset=utf-8" http-equiv="Content-Type"/>
  </head>
  <body>
    <!-- Replace width and height with the desired values -->
    <canvas id="amethyst-canvas" width="400" height="300" style="background: #ddddff;"></canvas><br />

    <!-- Include the JS generated by `wasm-pack build` -->
    <script src="spirv_cross/spirv_cross_wrapper_glsl.js"></script>
    <script src='pkg/myapp.js'></script>

    <script src="js/cpal.js"></script>

    <!-- Get the user to execute an action, otherwise audio will not play. -->
    <button>Launch MyApp</button>

    <script type=module>
      async function run() {
        const module = window.sc_internal_wrapper().then(async module => {
          window.sc_internal = module;

          await wasm_bindgen('./pkg/myapp_bg.wasm');

          let canvas = document.getElementById("amethyst-canvas");

          // The WASM application should not send synchronous requests on the
          // main thread, so we load the files and pass them in.
          let input_bindings = await fetch('config/input.ron')
            .then((response) => { return response.text(); });
          console.log(input_bindings);

          wasm_bindgen.MyAppBuilder
            .new()
            .with_canvas(canvas)
            .with_input_bindings(input_bindings)
            .run();
        });
      }

      // run();
      const launchBtn = document.querySelector('button:nth-of-type(1)');
      launchBtn.onclick = function() {
        run();
      };
    </script>

  </body>
</html>
```

</div>

</details>

<details open>

<summary>2. Create <code>build_release.sh</code>.</summary>

<div style="padding-left: 40px;">

This is used to compile the application to a WASM binary.

***Important:*** Replace `myapp` with your project's crate name.

```bash
#!/bin/sh

set -ex

# A few steps are necessary to get this build working which makes it slightly
# nonstandard compared to most other builds.
#
# * First, the Rust standard library needs to be recompiled with atomics
#   enabled. to do that we use Cargo's unstable `-Zbuild-std` feature.
#
# * Next we need to compile everything with the `atomics` and `bulk-memory`
#   features enabled, ensuring that LLVM will generate atomic instructions,
#   shared memory, passive segments, etc.
#
# * Finally, `-Zbuild-std` is still in development, and one of its downsides
#   right now is rust-lang/wg-cargo-std-aware#47 where using `rust-lld` doesn't
#   work by default, which the wasm target uses. To work around that we find it
#   and put it in PATH

RUSTFLAGS='-C target-feature=+atomics,+bulk-memory' \
  cargo build --target wasm32-unknown-unknown -Z build-std=std,panic_abort \
  --features "wasm,gl" --release

# Note the usage of `--no-modules` here which is used to create an output which
# is usable from Web Workers. We notably can't use `--target bundler` since
# Webpack doesn't have support for atomics yet.
wasm-bindgen target/wasm32-unknown-unknown/release/myapp.wasm \
  --out-dir pkg --no-modules

# worker.js crashes because it does not have `AudioContext` / `webkitAudioContext` in scope.
# This prevents it from crashing.
audio_context_workaround="const lAudioContext = (typeof AudioContext !== 'undefined' ? AudioContext : typeof webkitAudioContext !== 'undefined' ? webkitAudioContext : null)"

sed -i "s/const lAudioContext.\+\$/${audio_context_workaround}/" 'pkg/myapp.js'
```

</div>

</details>

<details open>

<summary>3. Create <code>worker.js</code>.</summary>

<div style="padding-left: 40px;">

This is used to instantiate `Worker`s to pass to `rayon`.

***Important:*** Replace `myapp` with your project's crate name.

```js
// synchronously, using the browser, import out shim JS scripts
importScripts('pkg/myapp.js');

// Wait for the main thread to send us the shared module/memory. Once we've got
// it, initialize it all with the `wasm_bindgen` global we imported via
// `importScripts`.
//
// After our first message all subsequent messages are an entry point to run,
// so we just do that.
self.onmessage = event => {
  let initialised = wasm_bindgen(...event.data).catch(err => {
    // Propagate to main `onerror`:
    setTimeout(() => {
      throw err;
    });
    // Rethrow to keep promise rejected and prevent execution of further commands:
    throw err;
  });

  self.onmessage = async event => {
    // This will queue further commands up until the module is fully initialised:
    await initialised;
    wasm_bindgen.child_entry_point(event.data);
  };
};
```

</div>

</details>

<details open>

<summary>4. Create <code>js/cpal.js</code>.</summary>

<div style="padding-left: 40px;">

```js
function copy_audio_buffer(dest, src, channel) {
    // Turn the array view into owned memory.
    var standalone = [...src];
    // Make it a Float32Array.
    var buffer = new Float32Array(standalone);

    // Copy the data.
    dest.copyToChannel(buffer, channel);
}

if (typeof exports === 'object' && typeof module === 'object')
    module.exports = copy_audio_buffer;
else if (typeof define === 'function' && define['amd'])
    define([], function() { return copy_audio_buffer; });
else if (typeof exports === 'object')
    exports["copy_audio_buffer"] = copy_audio_buffer;
```

</div>

</details>

<details open>

<summary>5. Create <code>js/web_socket_send.js</code> if your project uses functionality from <code>amethyst_network</code>.</summary>

<div style="padding-left: 40px;">

```js
function web_socket_send(web_socket, src) {
    // Turn the array view into owned memory.
    var standalone = [...src];
    // Make it a Uint8Array.
    let bytes = new Uint8Array(standalone);

    web_socket.send(bytes);
}

if (typeof exports === 'object' && typeof module === 'object')
    module.exports = bytes_owned;
else if (typeof define === 'function' && define['amd'])
    define([], function() { return bytes_owned; });
else if (typeof exports === 'object')
    exports["bytes_owned"] = bytes_owned;
```

</div>

</details>

The next page covers compiling and running the project as a WASM application,