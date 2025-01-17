# JS Open Serialization Scheme (JOSS)

Serialize JavaScript data in an open binary format to seamlessly exchange structured data between JavaScript runtime environments.
Compatible with browsers, Deno, and Node.js.
* Serializes almost every intrinsic JavaScript data type and data structure, including those not native to JSON, such as ArrayBuffer, BigInt, Date, Map, RegExp, Set, and TypedArray.
* Serializes primitive wrapper objects, sparse arrays, signed zeros, and circular references.
* Supports serializing to readable streams.

# Downloads

| Type | Environment | Filename |
| :--- | :--- | :--- |
| ES module | Browsers, Deno | [joss.min.js](https://github.com/quantirisk/joss/raw/main/joss.min.js) |
| CommonJS module | Node.js | [joss.node.min.js](https://github.com/quantirisk/joss/raw/main/joss.node.min.js) |
| Immediately Invoked Function Expression | Less recent browsers | [joss.iife.min.js](https://github.com/quantirisk/joss/raw/main/joss.iife.min.js) |

This project is licensed under the [MIT License](LICENSE.md).


# Methods
Use the [`serialize`](#serializedata-options) and [`deserialize`](#deserializebytes) methods for serializations in the form of static data.
Use the [`serializable`](#serializabledata-options) and [`deserializable`](#deserializableoptions) methods for serializations in the form of readable streams.
Use the [`deserializing`](#deserializingreadable-options) method for an alternative way to deserialize readable streams.

## serialize(data, [options])
* **data** `<any>`\
  The data to be serialized.
* **options** `<Object>`\
  An optional object that includes the following properties:
    * **endian** `<string>`\
      The endianness of TypedArrays in the serialization. Accepted values are `"LE"` for little-endian and `"BE"` for big-endian. If the source and target machines have different endianness, setting this property to the endianness of the slower machine ensures that the swapping of byte orderings happens on the faster machine. Defaults to the endianness of the source machine.
* **Returns** `<Uint8Array>` | `<Buffer>` in Node.js\
  The serialized bytes.


## deserialize(bytes)
* **bytes** `<Uint8Array>` | `<Buffer>` in Node.js\
  The serialized bytes.
* **Returns** `<any>`\
  The deserialized data.


## serializable(data, [options])
* **data** `<any>`\
  The data to be serialized.
* **options** `<Object>`\
  See the options parameter for `serialize`.
* **Returns** `<ReadableStream>` | `<stream.Readable>` in Node.js\
  A readable stream from which the serialized bytes can be read.


## deserializable([options])
* **options** `<Object>`\
  An optional object that includes the following properties:
  * **maxlength** `<number>`\
    The maximum acceptable length of the serialized bytes. Must be a positive integer. Defaults to 1GB.
* **Returns** `<WritableStream>` | `<stream.Writable>` in Node.js\
  A writable stream to which the serialized bytes can be written.
  After the writing process has completed successfully, the deserialized data can be accessed from the custom **`result`** property.


## deserializing(readable, [options])
* **readable** `<ReadableStream>` | `<stream.Readable>` in Node.js | `<AsyncIterable>`\
  A readable stream or async iterable object from which the serialized bytes can be read.
* **options** `<Object>`\
  See the options parameter for `deserializable`.
* **Returns** `<Promise>`\
  A promise that resolves to the deserialized data.


# Examples

## Deep Clone
The following is an example of deep cloning in Node.js.
The example is intended to illustrate the syntax of the serialization and deserialization methods.
```javascript
  const JOSS = require("/path/to/joss.node.min.js");     // Import the module.
  const data = { foo : { bar: "baz" } };                 // Define the data to be serialized.

  const bytes = JOSS.serialize(data);                    // Clone using serialize and
  const copy = JOSS.deserialize(bytes);                  // deserialize.
  console.log(data, copy);

  const readable = JOSS.serializable(data);              // Clone using serializable and
  const writable = JOSS.deserializable();                // deserializable by piping the
  readable.pipe(writable).on("finish", () => {           // streams. However, this is not
    const copy = writable.result;                        // supported in Firefox and Safari.
    console.log(data, copy);
  });

  const readable2 = JOSS.serializable(data);             // Clone using serializable and
  const promise = JOSS.deserializing(readable2);         // deserializing. This has
  promise.then((result) => {                             // better browser support.
    const copy = result;
    console.log(data, copy);
  });
```


## Fetch API
The following is an example of a HTTP request made using the `Fetch API`. The serialization and deserialization methods are determined by feature detection.
```javascript
  import * as JOSS from "/path/to/joss.min.js";
  const data = { foo : { bar: "baz" } };
  const options = { method: "POST", headers: { "Content-Type": "application/octet-stream" } };
  if (new Request("", {method: "POST", body: new ReadableStream()}).headers.has("Content-Type")) {
    options.body = JOSS.serialize(data);                          // Call serialize
  } else {
    options.body = JOSS.serializable(data);                       // Call serializable
  }
  fetch("/path/to/resource", options).then(async function(response) {
    if (response.body === undefined) {
      const buffer = await response.arrayBuffer();
      const bytes = new Uint8Array(buffer);
      const data = JOSS.deserialize(bytes);                       // Call deserialize
    } else if (response.body.pipeTo === undefined) {
      const data = await JOSS.deserializing(response.body);       // Call deserializing
    } else {
      const writable = JOSS.deserializable();                     // Call deserializable
      await response.body.pipeTo(writable);
      const data = writable.result;
    }
  });
```

The option to assign a `ReadableStream` to the body of a `Fetch` request is available in Chromium-based browsers as an experimental feature at the time of writing.
It can be enabled by navigating to `chrome://flags` and activating the `enable-experimental-web-platform-features` flag.
Please see [this page](https://web.dev/fetch-upload-streaming/#feature-detection) for more information about request streams.

## Deno HTTP Server
The following is an example of a HTTP server in Deno.
Incoming data is deserialized using the [`deserializing`](#deserializingreadable-options) method.
Outgoing data is serialized using the [`serializable`](#serializabledata-options) method.
```javascript
  import { serializable, deserializing } from "/path/to/joss.min.js";
  import { listenAndServe } from "https://deno.land/std/http/mod.ts";
  import { readerFromStreamReader } from "https://deno.land/std/io/mod.ts";
  import { iter } from "https://deno.land/std/io/util.ts";
  listenAndServe({hostname: "127.0.0.1", port: 8080}, async (request) => {
    const stream = iter(request.body);                            // Deno.Reader to AsyncIterable
    const data = await deserializing(stream);                     // Call deserializing
    // ...
    const readable = serializable(data);                          // Call serializable
    const reader = readerFromStreamReader(readable.getReader());  // ReadableStream to Deno.Reader
    request.respond({body: reader});
  });
```


## Node.js HTTP Server
The following is an example of a HTTP server in Node.js.
Incoming data is deserialized using the [`deserializable`](#deserializableoptions) method.
Outgoing data is serialized using the [`serializable`](#serializabledata-options) method.

```javascript
  const { serializable, deserializable } = require("/path/to/joss.node.min.js");
  const { createServer } = require("http");
  createServer((request, response) => {
    const writable = deserializable();                            // Call deserializable
    writable.on("finish", () => {
      const data = writable.result;
      // ...
      const readable = serializable(data);                        // Call serializable
      readable.pipe(response);
    });
    request.pipe(writable);
  }).listen(8080, "127.0.0.1");

```


## XMLHttpRequest
The following is an example of an AJAX request.
Outgoing data is serialized using [`JOSS.serialize`](#serializedata-options), which is analogous to `JSON.stringify`.
Incoming data is deserialized using [`JOSS.deserialize`](#deserializebytes), which is analogous to `JSON.parse`.
```javascript
  const data = { foo : { bar: "baz" } };
  const request = new XMLHttpRequest();
  request.open("POST", "/path/to/resource", true);
  request.responseType = "arraybuffer";
  request.setRequestHeader("Content-Type", "application/octet-stream");
  request.onload = function() {
    const bytes = new Uint8Array(request.response);
    const data = JOSS.deserialize(bytes);  // JOSS.deserialize is analogous to JSON.parse
    // ...
  };
  request.send(JOSS.serialize(data));      // JOSS.serialize is analogous to JSON.stringify
```

The `JOSS` variable is included in the global namespace using an immediately invoked function expression (IIFE).
```html
  <script defer nomodule src="/path/to/joss.iife.js"></script>
```

# Supported Types
The serialization format supports the following data types and data structures:
* `null`
* `undefined`
* `Boolean` (including primitive wrapper object)
* `BigInt` (including primitive wrapper object)
* `Number` (including primitive wrapper object, `Infinity`, `-Infinity`, `NaN`, and `-0`)
* `String` (including primitive wrapper object)
* `ArrayBuffer`
* `SharedArrayBuffer`
* `TypedArray` (including `BigInt64Array` and `BigUint46Array`)
* `DataView`
* `Array` (dense and sparse)
* `Object`
* `Map`
* `Set`
* `Date`
* `RegExp`
* Object references
* Custom objects

Please see the [official specification](SPECS.md) for details.
