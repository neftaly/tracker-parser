# tracker-parser
Converts a pr-tracker log to a list of events or states.

## usage
```js
import {
  parser,
  states
} from 'tracker-parser';

const events = parser(log);
const history = states(undefined, events);

console.log(
  'Last state',
  history.get(-1).toJS()
);
```
## `parser`
` :: logfile -> [...events]`

Using schema, converts log file from a Uint8Array, Array, or String,
to an array of events.

Values with decoding errors are set to `null`.

## `states`
` :: [...initial] || undefined => [...events] -> [...initial, ...new]`

Converts a List of initial states (or undefined) and an Array of events to a
List of states, separated by tick.

Lists use [immutable.js](https://facebook.github.io/immutable-js/).

The most common immutable List operations are:
  * `.get(index)` for a particular tick (index -1 for last)
  * `.toJS()` to convert to a plain JS object.

## `statesStream`
` :: [...initial] || undefined => [...events] => cb -> (Promise -> states)`

Same as `states`, except events are processed in preemtable chunks,
and result is returned as a promise.

`cb(done, states)` is fired on every chunk, with 2 arguments:
  * **done**: usually false, true on last chunk
  * **states**: interim List of tick states

## Message schema

[schema.json](./src/schema.json) contains a map of message IDs,
and the associated decoding instructions.

```json
{
  "32": {
    "type": "vehicleUpdate",
    "multiple": true,
    "format": [
      [ "flags", "bitmask", 1 ],
      [ "id", "int16" ],
      [ "status", "conditional", "flags", [
        [ "team", "int8" ],
        [ "position", "collection", [
          [ "x", "int16" ],
          [ "y", "int16" ],
          [ "z", "int16" ]
        ] ],
        [ "yaw", "int8" ],
        [ "health", "int16" ]
      ] ]
    ]
  },
  ...
}
```

### `type`: **string**
Message type name.
Passed on directly to the parsed object.

### `comment` *optional*: **string** OR **array**
Comments regarding a particular message.
For multiple lines, use an array of strings.

### `multiple` *optional*: **boolean**
Whether a message can contain multiple sets of data.

  * `true`: message will be continually parsed until the entire chunk is read
  * `false`: message data beyond that defined in `format` will be ignored

### `format`: **array**
An list of `[ key, type, ... ]` tuples, defining data format and order.

Valid types are:
  * **int8**: 1 byte signed integer
  * **uint8**: 1 byte unsigned integer
  * **int16**: 2 byte signed integer
  * **uint16**: 2 byte unsigned integer
  * **int32**: 4 byte signed integer
  * **uint32**: 4 byte unsigned integer
  * **float32**: 4 byte float
  * **float64**: 8 byte float
  * **string**: null-terminated string
  * **bool**: 1 byte boolean
  * **static**: 0 byte static value
  * **1uint7**: 1 byte concatenation of 1-bit boolean & 7-bit number
  * **group**: Same as *1uint7*, except first value becomes integer+1
  * **collection**: nested group of values
  * **bitmask**: group of true/false bytes
  * **condCollection**: nested group of values, read depending on first value
  * **conditional**: nested group of values, read depending on prior `bitmask`

Number types are expected to be little-endian.

#### `static`
Static value. 0 bytes will be read from buffer.
Third argument will be returned instead.

#### `1uint7`
Concatenated 1-bit boolean and 7-bit number.
The third argument contain a list of keys for each value.

```js
[ key, "1uint7", [
  ...keys
] ]
```

#### `group`
Same as a `1uint7`, except that the first value is an integer,
which gets incremented by 1.

#### `collection`
A collection is a nested list of values.
The third argument contains a nested list of type definitions.

```js
[ key, "collection", [
  ...types
] ]
```

#### `bitmask`
A bitmask is a list of bit values.
The third argument indicates the bitmask length in bytes.

```js
[ key, "bitmask", length ]
```

#### `condCollection`
**Note**: WIP

Same as a collection,
except that values after the first are only read if first is > 0;

```js
[ key, "collection", [
  firstType,
  ...types
] ]
```

#### `conditional`
**Note**: WIP

A conditional is a nested group of values,
that are turned on or off depending on a prior `bitmask`.

The third argument contains the key of the prior `bitmask`,
and the fourth contains a list of type definitions.

```js
[ key, "conditional", conditionalKey, [
  ...values
] ]
```
