# Simple File System and Modules

In addition to executing individual JavaScript files, ckb-js-vm also supports JavaScript modules through its Simple File
System. Files within this file system are made available for JavaScript to read, import, and execute, enabling module
imports like `import { * } from "./module.js"`. Each Simple File System must contain at least one entry file named
`index.bc` (or `index.js`), which ckb-js-vm loads from any cell and executes.

A file system is represented as a binary file with a specific format described in this document. You can use the
[ckb-fs-packer](https://github.com/nervosnetwork/ckb-js-vm/tree/main/packages/fs-packer) tool to create a file system from
your source files or to unpack an existing file system.

## How to create a Simple File System

Consider the following two files:

```js
// File index.js
import { fib } from "./fib_module.js";
console.log("fib(10)=", fib(10));
```

```js
// File fib_module.js
export function fib(n) {
    if (n <= 0)
        return 0;
    else if (n == 1)
        return 1;
    else
        return fib(n - 1) + fib(n - 2);
}
```
If we want ckb-js-vm to execute this code smoothly, we must package them into a
file system first. To pack them within the current directory into `fib.fs`, you
may run
```shell
npx ckb-fs-packer pack fib.fs index.js fib_module.js
```

Note that all file paths provided to `fs-packer` must be in relative path format. The absolute path of a file in your
local filesystem is usually meaningless within the Simple File System.

You can also rename files when adding them to the filesystem by using the `source:destination` syntax:

  ```shell
  npx ckb-fs-packer pack archive.fs 1.js:lib/1.js 2.js:lib/2.js
  ```

In this example, the local files `1.js` and `2.js` will be stored in the Simple File System as `lib/1.js` and
`lib/2.js` respectively.

## How to deploy and use Simple File System

While it's often more resource-efficient to write all JavaScript code in a single file, you can enable file system
support in ckb-js-vm through either:
- Executing or spawning ckb-js-vm with the "-f" parameter
- Using ckb-js-vm flags with file system enabled (see the [Working with ckb-js-vm](./ckb-js-vm.md) chapter for details)

## Unpacking a Simple File System

To extract files from an existing file system, run:
```shell
npx ckb-fs-packer unpack fib.fs .
```

## Simple File System On-disk Representation

The on-disk representation of a Simple File System consists of three parts:

1. A file count: A number representing the total files contained in the file system
2. Metadata array: Stores information about each file's name and content
3. Payload array: Binary objects (blobs) containing the actual file contents

Each metadata entry contains offset and length information for both a file's name and content. For each file, the
metadata stores four `uint32_t` values:
- The offset of the file name in the payload array
- The length of the file name
- The offset of the file content in the payload array
- The length of the file content

We can represent these structures using C-like syntax:
```c
struct Blob {
    uint32_t offset;
    uint32_t length;
}

struct Metadata {
    struct Blob file_name;
    struct Blob file_content;
}

struct SimpleFileSystem {
    uint32_t file_count;
    struct Metadata metadata[..];
    uint8_t payload[..];
}
```

When serializing the file system into a file, all integers are encoded as a
32-bit little-endian number. The file names are stored as null terminated
strings.

## QuickJS Null Termination Workaround

Due to an [issue in QuickJS](https://github.com/bellard/quickjs/issues/176), JavaScript source code strings must be
null-terminated. To address this requirement, ckb-js-vm automatically adds a null byte (`\0`) to every file without
including it in the reported `length` value.

For example, consider this simple JavaScript code:
```javascript
console.log("hi")
```

While the content length is 17 characters, when cast to a C-style string (`const char*`), an additional `\0` character
is appended after the final `)` character. This ensures QuickJS can properly process the source code.

## Using init.bc/init.js Files

The ckb-js-vm supports special initialization files named `init.bc` or `init.js` that are loaded and executed before
`index.bc` or `index.js`. This feature helps solve issues related to JavaScript module hoisting.

Consider this example code:

```javascript
import * as bindings from "@ckb-js-std/bindings";
bindings.mount(2, bindings.SOURCE_CELL_DEP, "/")
import * as module from './fib_module.js';
```

Due to JavaScript's [hoisting behavior](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting), import statements
are processed before other code executes. The code effectively becomes:

```javascript
import * as bindings from "@ckb-js-std/bindings";
import * as module from './fib_module.js';

bindings.mount(2, bindings.SOURCE_CELL_DEP, "/")
```

This will fail because the import attempts to access `./fib_module.js` before the file system is mounted. To solve this
problem, place the `bindings.mount` statement in an `init.bc` or `init.js` file, which will execute before any imports are
processed in the main file.

