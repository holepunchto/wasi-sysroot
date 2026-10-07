# wasi-sysroot

Prebuilt WASI sysroots for the [`llvm-runtime`](https://github.com/holepunchto/llvm-runtime) compiler. wasi-libc and the C++ standard library are built from source for each supported target.

```
npm i wasi-sysroot
```

The C++ libraries are built with and without WebAssembly exception handling, and the compiler picks one based on `-fwasm-exceptions`.

## Usage

```js
const sysroot = require('wasi-sysroot')

sysroot('wasm32-wasip1')
// /path/to/node_modules/wasi-sysroot-wasm32-wasip1
```

## License

Apache-2.0
