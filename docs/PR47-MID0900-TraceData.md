# PR #47 — MID 0900: Trace Data Support

> **Pull Request**: [#47 — Add parsing for MID0900](https://github.com/st-one-io/node-open-protocol/pull/47)
> **Author**: alexmc1510
> **Merged locally**: 2025 (from upstream PR opened in 2023)
> **Version**: 1.1.2 (`node-open-protocol-extended`)

---

## Overview

This change adds complete support for **MID 0900 – Trace Data (Revision 1)** to the
`node-open-protocol` library. MID 0900 carries detailed curve/trace information
captured during a tightening process on Atlas Copco (and compatible) controllers.

Before this change, trace data had to be parsed manually from raw buffers.
With MID 0900 now implemented, the library can automatically parse and serialize
trace messages, making curve data available as structured JavaScript objects.

---

## What Is Trace Data?

During every tightening cycle the controller records time-series data for
physical quantities such as **torque**, **angle**, and **motor current**. These
curves are transmitted as MID 0900 messages and typically include:

| Information | Description |
|---|---|
| **Result ID** | Unique identifier linking the trace to a tightening result |
| **Timestamp** | Date/time when the measurement was taken |
| **Data Fields** | Structured parameter records (parameter ID, data type, unit, step number, value) |
| **Resolution Fields** | Sampling interval definitions (first/last index, length, data type, unit, time value) |
| **Trace Samples** | The actual curve data — an array of time/value pairs |
| **Trace Type** | Identifies the kind of trace |
| **Transducer Type** | Which transducer produced the data |
| **Unit** | Unit of measurement for the trace values |

---

## Files Changed

### New file — `src/mid/0900.js`

The core implementation of the MID 0900 handler. Exports three functions:

#### `parser(msg, opts, cb)`

Parses an incoming MID 0900 buffer into a structured payload object.

**Parsed fields (Revision 1):**

| Field | Type | Length | Description |
|---|---|---|---|
| `resultID` | number | 10 | Tightening result identifier |
| `timeStamp` | string | 19 | ISO-style timestamp of the measurement |
| `numberPID` | number | 3 | Number of parameter-ID data fields |
| `fieldPID` | array | variable | Array of PID data field objects |
| `traceType` | number | 2 | Type of trace |
| `transducerType` | number | 2 | Transducer type code |
| `unit` | string | 3 | Unit code (resolved to a name via constants) |
| `numberData` | number | 3 | Number of additional data fields |
| `fieldData` | array | variable | Array of additional data field objects |
| `numberResolution` | number | 3 | Number of resolution field entries |
| `resolutionFields` | array | variable | Array of resolution field objects |
| `numberTrace` | number | 5 | Total number of trace samples |
| `sampleTrace` | array | variable | Array of `{ timeStamp, value }` objects |

**Data Field structure** (each element of `fieldPID` / `fieldData`):

| Field | Type | Length | Description |
|---|---|---|---|
| `parameterID` | string | 5 | Parameter identifier (e.g. `"02213"`) |
| `parameterName` | string | — | Human-readable name from constants |
| `length` | number | 3 | Length of the data value |
| `dataType` | number | 2 | Data type code |
| `unit` | string | 3 | Unit code |
| `unitName` | string | — | Human-readable unit name |
| `stepNumber` | number | 4 | Step number in the tightening sequence |
| `dataValue` | string | variable | The actual value |

**Resolution Field structure** (each element of `resolutionFields`):

| Field | Type | Length | Description |
|---|---|---|---|
| `firstIndex` | number | 5 | First sample index |
| `lastIndex` | number | 5 | Last sample index |
| `length` | number | 3 | Length of the time value |
| `dataType` | number | 2 | Data type code |
| `unit` | string | 3 | Unit code (time unit, e.g. `"200"` = seconds) |
| `unitName` | string | — | Human-readable unit name |
| `timeValue` | number | variable | Sampling interval |

**Trace Sample structure** (each element of `sampleTrace`):

| Field | Type | Description |
|---|---|---|
| `timeStamp` | Date | Calculated timestamp for this sample |
| `value` | number | Measured value (scaled by the coefficient from `fieldData`) |

The sample values are read as 16-bit signed integers (big-endian hex), then
multiplied by a coefficient extracted from `fieldData` (parameter `02213` or `02214`).
The timestamp of each sample is computed by adding `timeValue × multiplier × sampleIndex`
to the base timestamp.

#### `serializer(msg, opts, cb)`

Builds an outgoing MID 0900 subscription message.

- **Acknowledgement mode** (`msg.isAck = true`): emits MID 0005 with payload `"0900"`.
- **Subscription mode (default)**: emits MID 0008 (subscribe).
  - If no explicit payload fields (`midNumber`, `dataLength`, `extraData`, `revision`)
    are provided, it **automatically subscribes to the last three default curves**
    (Angle, Torque, Current) using the hardcoded buffer:
    ```
    09000014100000000000000000000000000000003001002003
    ```
  - If fields are provided, a custom subscription buffer is built from:
    `extraData` + `dataLength` + `revision` + `midNumber` (serialised in reverse order
    into a correctly sized buffer).

#### `revision()`

Returns `[1]` — only Revision 1 is currently supported.

---

### Modified — `src/helpers.js`

Three new helper functions were added to support the complex nested payload
structure of MID 0900:

#### `processDataFields(message, buffer, parameter, count, position, cb)`

Extracts an array of **Data Field** structures from the buffer.
Each iteration reads: `parameterID` (5), `length` (3), `dataType` (2),
`unit` (3), `stepNumber` (4), and `dataValue` (variable length).
Parameter and unit codes are resolved to human-readable names via `constants.json`.

#### `processResolutionFields(message, buffer, parameter, count, position, cb)`

Extracts an array of **Resolution Field** structures from the buffer.
Each iteration reads: `firstIndex` (5), `lastIndex` (5), `length` (3),
`dataType` (2), `unit` (3), and `timeValue` (variable length).

#### `processTraceSamples(message, buffer, parameter, count, position, timeStamp, timeValue, unit, cb)`

Extracts the raw trace sample bytes and converts them into an array of
`{ timeStamp, value }` objects.

- Reads each sample as 2 bytes (hex), interprets as a signed 16-bit integer.
- Scales values by a coefficient derived from `fieldData` (PID `02213` → `1/value`,
  PID `02214` → direct value).
- Computes each sample's timestamp by offsetting from the base timestamp using
  the resolution's `timeValue` and a unit multiplier:
  | Unit code | Multiplier | Meaning |
  |---|---|---|
  | `200` | 1 000 | seconds → ms |
  | `201` | 60 000 | minutes → ms |
  | `202` | 1 | milliseconds |
  | `203` | 3 600 000 | hours → ms |

All three functions are exported from `helpers.js` and available via `require("node-open-protocol").helpers`.

#### `testNul(object, buffer, parameter, position, cb)`

Pre-existing helper, but now used by MID 0900's parser to verify the NUL
character separator between `numberTrace` and the trace sample data.

---

### Modified — `src/openProtocolParser.js`

A special-case offset adjustment was added for MID 0900 in the `_transform`
method. Because MID 0900 payloads do not include the trailing NUL character
in the same way as other MIDs, the pointer is decremented by 1 before
finalising the parsed object:

```js
if (obj.mid === 900) {
    ptr = ptr - 1;
}
```

This prevents an off-by-one error when slicing the raw buffer for MID 0900
messages.

---

### Modified — `src/midData.json`

Added the mapping:

```json
"900": "traceData"
```

This registers the human-readable name `"traceData"` for MID 900, used
throughout the library for event naming and logging.

---

### Modified — `src/midGroups.json`

Added the subscription group:

```json
"traceData": {
    "data": 900,
    "subscribe": 900,
    "unsubscribe": 9,
    "ack": 900,
    "generic": true
}
```

This enables the standard `subscribe("traceData")` / `unsubscribe("traceData")`
workflow via the `SessionControlClient`.

> **Note:** The subscription MID is `900` itself (the serializer converts it to
> MID 0008 internally). Unsubscribe uses MID `9` (MID 0009 —
> *Subscribe Data Acknowledge*).

---

### Modified — `CHANGELOG.md`

Added entry:

```
#### v1.1.1
 - Implemented parsing of MID 0900
```

---

### Modified — `package.json`

- **Name** changed to `node-open-protocol-extended`
- **Version** bumped to `1.1.2`
- **Author** updated to `SmartTech & Alejandro de la Mata`
- **License** changed to `GPL-3.0-or-later`
- **Repository URL** updated to the author's fork

---

### Modified — `index.js`

Copyright header updated to include Alejandro de la Mata Chico (2023).
No functional API changes — MID 0900 is automatically discovered by the
existing `helpers.getMids()` mechanism that scans `src/mid/*.js`.

---

## Usage Example

### Subscribing to trace data

```js
const { createClient } = require("node-open-protocol");

const client = createClient(4545, "192.168.1.100");

client.on("connect", () => {
    // Subscribe to trace data (Angle, Torque, Current by default)
    client.subscribe("traceData", (err) => {
        if (err) console.error("Subscribe error:", err);
    });
});

client.on("data", "traceData", (data) => {
    console.log("Result ID:", data.payload.resultID);
    console.log("Timestamp:", data.payload.timeStamp);
    console.log("Samples:",  data.payload.sampleTrace.length);

    // Each sample has { timeStamp: Date, value: number }
    data.payload.sampleTrace.forEach((sample) => {
        console.log(`  ${sample.timeStamp.toISOString()} => ${sample.value}`);
    });
});
```

### Acknowledging trace data

When you receive a trace data message, the library can send an acknowledgement
automatically (handled by the subscription mechanism), or you can do it
manually:

```js
client.sendMid({
    mid: 900,
    isAck: true
});
// Sends MID 0005 with payload "0900"
```

---

## Parsed Payload — Full Object Shape

```js
{
    mid: 900,
    revision: 1,
    payload: {
        resultID:         10000000001,        // number (10 digits)
        timeStamp:        "2023-06-15 14:30:00", // string (19 chars)
        numberPID:        2,                  // number
        fieldPID: [                           // array of data fields
            {
                parameterID:   "00021",
                parameterName: "Torque",
                length:        6,
                dataType:      1,
                unit:          "001",
                unitName:      "Nm",
                stepNumber:    1,
                dataValue:     "12.500"
            }
            // ... (numberPID entries)
        ],
        traceType:        1,                  // number
        transducerType:   1,                  // number
        unit:             "001",              // string
        unitName:         "Nm",               // string (resolved)
        numberData:       1,                  // number
        fieldData: [                          // array of data fields
            {
                parameterID:   "02213",
                parameterName: "Coefficient",
                length:        5,
                dataType:      7,
                unit:          "000",
                unitName:      "",
                stepNumber:    0,
                dataValue:     "100.0"
            }
            // ... (numberData entries)
        ],
        numberResolution: 1,                  // number
        resolutionFields: [                   // array of resolution fields
            {
                firstIndex: 0,
                lastIndex:  999,
                length:     5,
                dataType:   7,
                unit:       "200",
                unitName:   "s",
                timeValue:  0.001
            }
            // ... (numberResolution entries)
        ],
        numberTrace:      1000,               // number
        sampleTrace: [                        // array of trace samples
            {
                timeStamp: Date,              // JavaScript Date object
                value:     12.5               // number (scaled by coefficient)
            }
            // ... (numberTrace entries)
        ]
    }
}
```
