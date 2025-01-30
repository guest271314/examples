# Executing AssemblyScript directly, and compiling to JavaScript with tsc, Deno, and Bun (and executing WASM directly with bun)

# Installing 

```
bun install typescript@next assemblyscript @types/node
bun add v1.2.1-canary.15 (5633ec43)

installed typescript@5.8.0-dev.20250129 with binaries:
 - tsc
 - tsserver
installed assemblyscript@0.27.32 with binaries:
 - asc
 - asinit
installed @types/node@22.12.0

6 packages installed [17.07s]
```

Install AssemblyScript's WASI shim

```
bun install assemblyscript/wasi-shim
bun add v1.2.1-canary.15 (5633ec43)

installed @assemblyscript/wasi-shim@github:assemblyscript/wasi-shim#4399cff

1 package installed [1.92s]

```

# Compiling to WASM with asc

The script accepts two integers reflecting the factorial integer to create
an `Array` of `len` length, and the nth lexicographic permutation `n` of the
created `Array`. The file is then copied to `node_modules/assemblyscript/std`.

We read `len` and `n` either as `stdin` to WASM with WASI support or as arguments
passed to the runtime (JavaScript or WebAssembly).

We import AssemblyScript's `std/portable.js` and `node:process`, and include 
two `// @ts-ignore` comments to ignore AssemblyScript's `read()` which is not 
the same as `process.read()` or `fs.read()`; and `String.UTF8.decode()`, which
will be replaced in the JavaScript transformed/compiled version

```
import "./portable/index.js";
import process from "node:process";
// array_nth_permutation
// https://stackoverflow.com/a/34238979
export function array_nth_permutation(len: i32, n: i32): void { //Array<f64>
  let lex = n; // length of the set
  let b: i32[] = []; // copy of the set a.slice()
  for (let x: i32 = 0; x < len; x++) {
    b.push(x);
  }
  const res: i32[] = []; // return value, undefined
  let i: i32 = 1;
  let f: i32 = 1;

  // compute f = factorial(len)
  for (; i <= len; i++) {
    f *= i;
  }

  let fac = f;
  // if the permutation number is within range
  if (n >= 0 && n < f) {
    // start with the empty set, loop for len elements
    // let result_len = 0;
    for (; len > 0; len--) {
      // determine the next element:
      // there are f/len subsets for each possible element,
      f /= len;
      // a simple division gives the leading element index
      i = (n - n % f) / f; // Math.floor(n / f);
      // alternately: i = (n - n % f) / f;
      // res[(result_len)++] = b[i];
      // for (let j = i; j < len; j++) {
      //   b[j] = b[j + 1]; // shift elements left
      // }
      res.push(<i32>b.splice(i, 1)[0]);
      // reduce n for the remaining subset:
      // compute the remainder of the above division
      n %= f;
      // extract the i-th element from b and push it at the end of res
    }

    let result: string = `[${res}]`;

    process.stdout.write(
      `${lex} of ${fac - 1} (0-indexed, factorial ${fac}) => ${result}\n`,
    );

    process.exit(0);
  } else {
    if (n === 0) {
      process.stdout.write(`${n} = 0`);
    }
    process.stdout.write(`${n} >= 0 && ${n} < ${f}: ${n >= 0 && n < f}`);
    process.exit(1);
  }
}

let input: string = "0";
let lex: string = "0";

if (process.argv.length > 1) {
  input = process.argv.at(-2);
  lex = process.argv.at(-1);
} else {
  let stdin = process.stdin;
  let buffer = new ArrayBuffer(64);
  // @ts-ignore
  let n: number = stdin.read(buffer);
  if (n > 0) {
    // @ts-ignore
    let data = String.UTF8.decode(buffer);
    input = data.slice(0, data.indexOf(" "));
    lex = data.slice(data.indexOf(" "), data.length);
  }
}

input = input.trim();
lex = lex.trim();

if (<i32> parseInt(input) < 2 || <i32> parseInt(lex) < 0) {
  process.stdout.write(`Expected n > 2, m >= 0, got ${input}, ${lex}`); // eval(input)
  process.exit(1);
}

array_nth_permutation(<i32> parseInt(input), <i32> parseInt(lex));
```

```
node_modules/.bin/asc --enable simd --exportStart --config \
./node_modules/@assemblyscript/wasi-shim/asconfig.json  module.ts -o module.wasm
```

# Test and verify the compiled WASM works as expected

```
wasmtime module.wasm 7 8
8 of 5039 (0-indexed, factorial 5040) => [0,1,2,4,5,3,6]
```

```
echo '9 9' | wasmer module.wasm
9 of 362879 (0-indexed, factorial 362880) => [0,1,2,3,4,6,7,8,5]
```

# Compile AssemblyScript to JavaScript with TypeScript's tsc

Create a symbolic link to `@types/node` in the same `node_modules` folder in 
`assemblyscript/std/types`

```
ln -sf "$PWD/node_modules/@types/node" "$PWD/node_modules/assemblyscript/std/types"
```

Create `tsconfig.json` file

```
touch tsconfig.json
```

Edit the file to include 

```
{
  "extends": "../tsconfig-base.json",
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "allowJs": true,
    "esModuleInterop": true,
    "typeRoots": [ "types" ],
    "types": [ "portable", "node" ],
    "lib": ["esnext", "esnext.string"]
  },
  "files": ["module.ts"],
  "fmt": {
    "useTabs": false,
    "lineWidth": 80,
    "indentWidth": 2
  }
}
```

Copy `tsconfig.json` to `assemblyscript/std`
```
cp tsconfig.json node_modules/assemblyscript/std
```

```
./node_modules/.bin/tsc -p ./node_modules/assemblyscript/std/tsconfig.json
```

creates `module.js` in `assemblyscript/std`.

# Modifying emitted JavaScript

Modify JavaScript emitted by `tsc`, `deno`, and `bun` to substitute `node:fs` 
`readSync()` for AssemblyScript's `read()`, `new Uint8Array(new ArrayBuffer(64))` 
for `ArrayBuffer(64)`, and `TextDecoder()` for 
`String.UTF8.decode()`, and comment `import ./portable/index.js`

```
// import"./portable/index.js";
import process from "node:process";
import { readSync } from "node:fs";
export function array_nth_permutation(len, n) {
  let lex = n;
  let b = [];
  for (let x = 0;x < len; x++) {
    b.push(x);
  }
  const res = [];
  let i = 1;
  let f = 1;
  for (;i <= len; i++) {
    f *= i;
  }
  let fac = f;
  if (n >= 0 && n < f) {
    for (;len > 0; len--) {
      f /= len;
      i = (n - n % f) / f;
      res.push(b.splice(i, 1)[0]);
      n %= f;
    }
    let result = `[${res}]`;
    process.stdout.write(`${lex} of ${fac - 1} (0-indexed, factorial ${fac}) => ${result}
`);
    process.exit(0);
  } else {
    if (n === 0) {
      process.stdout.write(`${n} = 0`);
    }
    process.stdout.write(`${n} >= 0 && ${n} < ${f}: ${n >= 0 && n < f}`);
    process.exit(1);
  }
}
let input = "0";
let lex = "0";
if (process.argv.length >= 3) {
  input = process.argv.at(-2);
  lex = process.argv.at(-1);
} else {
  let stdin = process.stdin;
    let buffer = new Uint8Array(new ArrayBuffer(64));
    // @ts-ignore
    // Use fs.readSync()
    let n = readSync(stdin.fd, buffer);
    // let n = stdin.read(64);
    if (n > 0) {
      // @ts-ignore
      // let data = String.UTF8.decode(buffer);
      // Use TextDecoder
      let data = new TextDecoder().decode(buffer);
      input = data.slice(0, data.indexOf(" "));
      lex = data.slice(data.indexOf(" "), data.length);
  }
}
input = input.trim();
lex = lex.trim();
if (parseInt(input) < 2 || parseInt(lex) < 0) {
  process.stdout.write(`Expected n > 2, m >= 0, got ${input}, ${lex}`);
  process.exit(1);
}
array_nth_permutation(parseInt(input), parseInt(lex));
```

# Deno

## Run AssemblyScript directly with Deno


Run AssemblyScript source code in `.ts` file directly with `deno`
```
deno -A -q -c node_modules/assemblyscript/std/tsconfig.json ./node_modules/assemblyscript/std/module.ts 12 2
2 of 479001599 (0-indexed, factorial 479001600) => [0,1,2,3,4,5,6,7,8,10,9,11]
```

## Compile AssemblyScript to JavaScript and cache with deno install
```
deno install -f -c node_modules/assemblyscript/std/tsconfig.json module.ts
Unsupported compiler options in "file:///home/user/node_modules/assemblyscript/std/tsconfig.json".
  The following options were ignored:
    allowJs, esModuleInterop, module, target, typeRoots
```

Write the cached file to current working directory

```
cat  "$HOME/.cache/deno/gen/file$PWD/module.ts.js" > module.ts.js
```

## Execute JavaScript compiled and cached by Deno from AssemblyScript source

```
deno -A module.ts.js 11 22
22 of 39916799 (0-indexed, factorial 39916800) => [0,1,2,3,4,5,6,10,9,7,8]

```


# Bun

`bun` runs `.wasm` files [Bun.build](https://bun.sh/docs/bundler)

> `.node` `.wasm`	These files are supported by the Bun runtime, but during bundling 
they are treated as [assets](https://bun.sh/docs/bundler#assets).

## Execute AssemblyScript complied to WASM directly with Bun
```
bun module.wasm 4 5
5 of 23 (0-indexed, factorial 24) => [0,3,2,1]
```

## Execute AssemblyScript directly with Bun

```
bun run node_modules/assemblyscript/std/module.ts 12 2
2 of 479001599 (0-indexed, factorial 479001600) => [0,1,2,3,4,5,6,7,8,10,9,11]
```

## Bundle AssemblyScript to JavaScript with bun build 

`--no-bundle` removes comments. Omit `--no-bundle` option to include `portable/index.js` in the bundle 

```
bun build node_modules/assemblyscript/std/module.ts --no-bundle --outfile module.js

  module.js  1.39 KB

[10ms] transpile

```

```
echo '4 5' | bun module.js
5 of 23 (0-indexed, factorial 24) => [0,3,2,1]
```

# Try executing AssemblyScript's `.ts` directly with node

```
node node_modules/assemblyscript/std/module.ts 12 2
node:internal/modules/typescript:170
    throw new ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING(filename);
          ^

Error [ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING]: Stripping types is currently 
unsupported for files under node_modules, for "file:///home/user/node_modules/assemblyscript/std/module.ts"
    at stripTypeScriptModuleTypes (node:internal/modules/typescript:170:11)
    at ModuleLoader.<anonymous> (node:internal/modules/esm/translators:547:16)
    at #translate (node:internal/modules/esm/loader:473:12)
    at ModuleLoader.loadAndTranslate (node:internal/modules/esm/loader:520:27)
    at async ModuleJob._link (node:internal/modules/esm/module_job:115:19) {
  code: 'ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING'
}

Node.js v24.0.0-nightly20250128532fff6b27

```

# Execute JavaScript compiled from AssemblyScript with deno install

```
node  module.ts.js 11 22
22 of 39916799 (0-indexed, factorial 39916800) => [0,1,2,3,4,5,6,10,9,7,8]
```
