JSON-Mmap: A Specification for JSON/binary JSON memory-map (mmap) for fast file IO
============================================================

- **Status of this document**: This document is currently under development.
- **Copyright**: (C) Qianqian Fang (2022) <q.fang at neu.edu>
- **License**: Apache License, Version 2.0
- **Revision Number**: 1 (Draft 1)
- **Version Number**: v0.5
- **Abstract**:

> JSON and binary-JSON formats are ubiquitously supported but were designed largely
for storage of lightweight hierachical data. However, when accessing large data records
inside a JSON/binary-JSON file, file reading/writing can be slow. This specification
aims to define a lightweight memory-mapping (mmap) table to enable fast disk-mapped
file IO for JSON and binary JSON formats. This JSON-Mmap table can be embedded in the same
data file or in a separate file. Using the byte-offset/length information stored in the
JSON-Mmap allows to read or update particular data record without needing to parse or
overwrite the entire data file, leading to dramatic performance improvement.


## Table of Content

- [Introduction](#introduction)
    * [Background](#background)
    * [Overview](#overview)
- [Syntax](#syntax)
    * [Path string](#path-string)
    * [Significant and insignficant characters](#significant-and-insignficant-characters)
    * [Locator vector](#locator-vector)
    * [Metadata keys and values](#metadata-keys-and-values)
    * [Storage of the JSON-Mmap table](#storage-of-the-json-mmap-table)
- [Sample utilities of JSON-Mmap](#sample-utilities-of-json-mmap)
    * [Fast read-only data-level access](#fast-read-only-data-level-access)
    * [On-disk value replacement with equal or less bytes](#on-disk-value-replacement-with-equal-or-less-bytes)
    * [On-disk value replacement with more bytes](#on-disk-value-replacement-with-more-bytes)
- [Recommended File Specifiers](#recommended-file-specifiers)
- [Summary](#summary)



Introduction
------------

### Background


The JSON (JavaScript Object Notation) format has been ubiquitously supported among
today's web and native applications for efficient exchange of hierachical data.
It is human-readable, lightweight and fast to parse, thanks to its simple syntax.
JSON-compatible readers and writers are widely supported among nearly all programming
environments. Despite its proliferated use, JSON has limitations. JSON is highly efficient
when storing and exchanging lightweight data records, but when storing highly complex
hierachical data and large data records (such as large arrays), JSON parsers and writers
exhibit low file-IO efficiency. A commonly used technique, memory-mapped (**mmap**) file I/O,
for accelerating disk-based data access, can only be used for binary-files with fixed/rigid
structures, and are not generally applicable to JSON files.

To address the file-IO efficiency issue, a number of binary-JSON formats have been proposed,
including [BSON (Binary JSON)](http://bson.org), [MessagePack](https://msgpack.org),
[CBOR (Concise Binary Object Representation, [RFC 7049])](https://cbor.io),
[UBJSON (Universal Binary JSON)](http://ubjson.org) and [BJData (Binary JData)](https://neurojson.org/bjdata).
Using binary-JSON formats to store JSON-based hierachical data in strongly-typed binary
forms can significantly reduce file sizes, as well as reading/writing overheads.
However, when reading/writing large binary data records such as an N-dimensional (ND)
array (ND-arrays), binary-JSON parsers may still show inferior efficiency compared
to mmap-based binary file IO over traditional fixed-sized binary files (which lack of
the capability of extension compared to JSON and binary JSON).

### Overview

In this specification, we aim to define a lightweight JSON-based mmap-table structure that
can be embedded inside a JSON or binary JSON file or stored side-by-side along the JSON/binary
JSON file to drmatically enhance their file IO efficiencies. The JSON-Mmap table is lightweight
and follows a simple structure to record essential serialization data of key records, including
disk/memory linear byte-offset and length. Reading/writing JSON-Mmap table does not require
additional parser/writer because it must be stored/accessed using the same format as the
respective target data file (JSON or binary-JSON). Creating and utilizing JSON-Mmap records
can be particularly beneficial when using the combination of JSON/[BJData](https://neurojson.org/bjdata)
serializations with [JData annotation](https://neurojson.org/jdata) methods for storing and
exchange of large sized scientific data files such as medical imaging scans.

Syntax
------------------------

A JSON-Mmap must be written as an **array of arrays**, with each element being an
**array of minimum two elements**, for example, in the below form

```json
[
    [ "name1", value1 ],
    [ "name2", value2 ],
    ...
    [ "path1", [...]  ],
    [ "path2", [...]  ],
    ...
]
```

In each of the sub-arrays, 
- the first element must be a string; such string can be one of the two cases:
    - a string starting with ASCII letter `$` (`0x24` in hex) is referred to as a **path**,
      representing a [JSON-Path](https://github.com/json-path/JsonPath) like reference, 
      pointing towards a specific data record to the associated JSON/binary JSON file, and
    - any string that does not start with `$` is referred to as a **metadata key**, to store
      optional auxillary information related to the JSON-Mmap,
- when the first string is a **path**, the second element must be an array of numeric values, hereinafter
  referred to as the **locator vector**, containing various information regarding the byte-location of the
  serialized data record on the disk or memory buffer
- when the frist string is a **metadata key**, the second element denotes the value of the metadata,
  some recommended metadata key value formats are listed below.

### Significant and insignficant characters

In a JSON document, zero or more whitespaces can be inserted between data items without alternating
the data structure stored in the document. Four whitespace characters are permitted:

- space (` `): 0x20 in hex (ASCII code: 32)
- linefeed (`\n`): 0x0A in hex (ASCII code: 20)
- carrage return (`\r`): 0x0D in hex (ASCII code: 13)
- horizontal tab (`\t`): 0x09 in hex (ASCII code: 9)

We call the above whitespace characters as **insignificant characters** and all non-whitespace 
characters as **significant characters** in a JSON document.

Similarly, in a BJData/UBJSON document, zero or more `no-op` marker `N` can be inserted between data
items without alternating the data stored in such file. As a result, we call the `no-op` marker `N` as
**insignificant character** and all other bytes that are not `no-op` are called **significant characters**.

Other binary JSON formats such as CBOR and MessagePack do not support `no-op` markers or optional
whitespaces, therefore, all bytes in these files are considered signficiant characters.

### Path string

The format of the path string is inspired and largely similar to the syntax of
[JSON-Path](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html), where
the string must follow the below format

|      Notation      |                                                   Meaning                                                   |
|--------------------|-------------------------------------------------------------------------------------------------------------|
|`$`                 | the root object, must be the first letter of all paths                                                      |
|`$[i]`              | i=0,1,2,..., denoting the (i+1)th root-level object in an concatenated JSON (CJSON) or binary JSON document |
|`.`                 | child of the object on the left                                                                             |
|`[i]`               | i=0,1,2,..., denoting the (i+1)th child of an array on the left                                             |
|`.key` or `.['key']`| a named child with a string-typed name `key`; using `['key']` is required when `key` has `.` ,`[` or `]`    |

Note: JSON-Mmap path specifier does not support additional annotations in JSONPath, such as `..` or `@` operator

For example, for the below JSON (or JSON-like) object

```json
{
    "name": "Andy",
    "school": "Hood",
    "schedule": {
        "Monday": [8,12],
        "Tuesday": null,
        "Friday": {
            "AM": 9,
            "PM": [14.5, 15.5]
        }
    }
}
{
    "name": "Leo",
    "school": "Hood",
    "schedule": {
        "Wednesday": [10]
    }
}
```
the following 
|                   Path               |                  Reference Value                          |
|--------------------------------------|-----------------------------------------------------------|
|`$[0].name`                           | "Andy"                                                    |
|`$[0].schedule.Monday[0]`             | 8                                                         |
|`$[0].schedule.Friday.AM`             | 9                                                         |
|`$[0].schedule.Friday.PM`             | [14.5, 15.5]                                              |
|`$[0].schedule.Friday.PM[1]`          | 15.5                                                      |
|`$[0]['schedule']['Friday']['PM'][1]` | 15.5 (same as above)                                      |
|`$[1]`                                | the entire 2nd root-level object `{"name":"Leo",...}`     |
|`$[1].schedule`                       | the value of the `schedule` object `{"Wednesday": [10]}`  |


### Locator vector

The locator vector must be an array (can be empty). It should have the following form
```
[<start>, <length>, <whitespace-byte-length-after>, <whitespace-byte-length-before> ...]
```
where

- (recommended) the first number `<start>`, if present, must be an integer, denoting the byte position 
  (starting from 1) of the first significant character of the referenced value from the begining of 
  the referenced document, 
- (recommended) the second number `<length>`, if present, must be an intger, denoting the end-to-end byte-length
  of the referenced value between the first significant character and the last significant character
  (inclusive)
- (optional) the third number, `<whitespace-byte-length-before>`, if present, must be an integer, denoting the
  byte-lengt of all whitespaces (insignificant character) before the first signficiant character and
  after the separator mark `,` with the preceding objects, or a significant character of the enclosing parent object.
- (optional) the forth number, `<whitespace-byte-length-after>`, if present, must be an integer, denoting the
  byte-lengt of all whitespaces (insignificant character) after the last signficiant character and
  before the separator mark `,` with the following objects, or the next significant character of the parent object.
- additional elements in the locator vector are reserved for future extension of this specification

Both the optional `<whitespace-byte-length-before>` and `<whitespace-byte-length-after>` records are
designed for the purpose of **in-place on-disk value replacement** of the data records. See
[Sample utilities of JSON-Mmap](#sample-utilities-of-json-mmap) section for detailed discussions.

For example, for the below JSON string buffer
```json
1         11        21        31        41        51        61        71        81 (byte index)
1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 
{"name" :  "Andy" , "schedule": { "Mon": [ 10 , 14], "Tue": null, "Wed":10.5 } }
           ^--                  ^--      ^--                ^--         ^--    : starting positions
         ‾‾                    ‾        ‾ ‾    ‾           ‾                   : preceding whitespaces
```

the corresponding JSON-Mmap is
```json
[
    ["$",                [1, 80]],
    ["$.name",           [12, 6, 2]],
    ["$.schedule",       [33, 47, 1]],
    ["$.schedule.Mon",   [42, 10, 1]],
    ["$.schedule.Mon[1]",[49, 2, 1]],
    ["$.schedule.Tue",   [64, 4, 1]],
    ["$.schedule.Wed",   [73, 4]]
]
```
**Note**: a JSON-Mmap does not have to contain mappings of all data elements of a document; it may contain
only a selected subset of data elements.

Similarly, the above JSON document can also be stored as a [BJData](https://neurojson.org/bjdata/draft2)/
[UBJSON](http://ubjson.org/) buffer in the below form (here numbers are printed in ASCII form instead
of their binary forms; we adjusted the byte-index indicators accordingly to match their correct binary
lengths; the `name` section of the data are underscored for easy readability)

```
1         11        21        31          41          53        61        71    (byte index)
1 3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3  5  7 9 1 3 5 7 9   3 5 7 9 1 3 5 7 9 1 3 5 7 9 1 3
{U4nameSU4AndyU4schedule{U3Mon[i10i14]U3TueZU3Wedd10.5}}
 ‾‾‾‾‾‾       ‾‾‾‾‾‾‾‾‾‾ ‾‾‾‾‾        ‾‾‾‾‾ ‾‾‾‾‾
```
the corresponding JSON-Mmap is
```json
[
    ["$",                [1, 54]],
    ["$.name",           [8, 7]],
    ["$.schedule",       [25, 19]],
    ["$.schedule.Mon",   [31, 6]],
    ["$.schedule.Mon[1]",[34, 2]],
    ["$.schedule.Tue",   [42, 1]],
    ["$.schedule.Wed",   [48, 5]]
]
```

## Metadata keys and values

Optionally, one can add `["name", value]` pairs in the form of a 2-element array in the JSON-Mmap to
provide useful metadata information. In such case, the `"name"` string must not start with letter `$`.

To ensure high performance when using JSON-Mmap, it is highly recommended to only use simple value
forms, such as numbres or strings in the metadata entries.

In the below table, we show a list of metadata examples.

|                  Metadata Key                |                            Value                           |
|----------------------------------------------|------------------------------------------------------------|
|`["MmapVersion", "0.5"`                       | the JSON-Mmap specification version number                 |
|`["ReferenceFileName","dat.json"]`            | the original referenced data file name                     |
|`["ReferenceFileURI","https://.../data.json"]`| the original referenced data file URL                      |
|`["ReferenceFileBytes": 92021]`               | the byte size of the referenced data file                  |
|`["ReferenceFileSHA256", "0ACA987B..."]`      | a string storing the SHA256 hash key of the referenced file|
|`["Comment", "a user-defined comment"`        | a comment                                                  |
|`["MmapByteLength", 2560]`                    | the total byte-length of the mmap table                    |

The file location metadata such as `"ReferenceFileName"` and `"ReferenceFileURI"` are
only for information purposes only, and shall not be in conflict with the 

### Storage of the JSON-Mmap table

The JSON-Mmap table can be stored in 3 possible locations

#### Inline direct form

The JSON-Mmap can be directly stored as the header of the referenced JSON/binary
JSON data in the form of a concatenated JSON object. An example is shown below

```json
[
    ["MmapVersion", "0.5"],
    ["Comment", "mmap for data1"],
    ["$",     [...]],
    ["$.key", [...]],
    ...
]{
    data1
}
```

An inline JSON-Mmap is automatically assumed to be associated with the concatenated
JSON object immediately following the mmap table; the `start` record in
all **locator vector** assumes the start of the referenced data buffer immediately
follows the last signficant character of the JSON-Mmap (in the above case,
the closing bracket `]`).

When whitespaces are inserted between the JSON-Mmap and the following JSON object,
such as new-lines `\n` or spaces, the length of such padded bytes must be considered
in the `start` records.

Multiple inline JSON-Mmap tables can be used in the same document, each preceding
its associated JSON object
```json
[["Comment", "mmap for data1"], ["$", [...]],...]{data1}
[["Comment", "mmao for data2"], ["$", [...]],...]{data2}
```

#### Inline embedded form

When a JSON-Mmap table is not directly stored as a root-level object, such as shown in the
below example as a sub-element of a root-level object `"_DataInfo_"`, the JSON-Mmap is again
assumed to be associated with the concatenated JSON object immediately following the last
significant character of the root-level container that encloses the mmap table (in this case,
the closing `}` of the `"_DataInfo_"` object).

```json
{
    "_DataInfo_": {
        "Creator": "NeuroJSON",
        "Comment": "JSON-Mmap table demo",
        "mmap": [
            ["MmapVersion", "0.5"],
            ["Comment", "mmap for data1"],
            ["$",     [...]],
            ["$.key", [...]],
            ...
        ]
    }
}{
    data1
}
```

Similar to the inline direct form, multiple embedded JSON-Mmap can be used in the same
document to reference to multiple subsequent root-level JSON objects.

#### Standalone form

A JSON-Mmap can be stored in a standalone file and stored separately from the associated
data file. A user or application shall apply the mmap to its matching associated data
file. It is suggested to use the `ReferenceFileSHA256` hash key, if present, to ensrure
the appropriate match of the mmap and the data.

Below, we show an example of a paired JSON-mmap and data files.

Standalone JSON-mmap file `data1.json.jmmap`:
```json
[
    ["MmapVersion", "0.5"],
    ["ReferenceFileName", "data1.json"],
    ["ReferenceFileSHA256", "099AC812..."],
    ["$",     [...]],
    ["$.key", [...]],
    ...
]
```
and the associated data file `data1.json`:
```
{
    data1
}
```

For a standalone JSON-Mmap file, there should be no significant character appearing after
the last significant character of the mmap record (if in direct form) or its enclosing
root-level container (if in embedded form).

Sample utilities of JSON-Mmap
------------------------------

When parsing a JSON or binary JSON file in which JSON-Mmap records are anticipated, an efficient
parser may take advanrage of the byte-offset/length information present in such data structure
to achieve high performance file reading and writing.

### Fast read-only data-level access

For simple files containing inline JSON-Mmap tables, the parser is recommended to read only the
first root-level object and identify whether it contains an mmap table. If an mmap table is found,
the parser can then utilize the individual data record information in mmap to achieve fast read-only
access to individual records without eneding to read/parse the entire referenced data buffer.

```
[
    ["$.key", ]
]{
    "key": 1024
}
```

A C implementation 


### On-disk value replacement with equal or less bytes


### On-disk value replacement with more bytes


Recommended File Specifiers
------------------------------

For JSON-based files containing either standalone or inline JSON-Mmap records, the
recommended file suffix is **`".jmmap"`**; for binary-JData (BJData) based files containing
either standalone or inline JSON-Mmap records, the recommended file suffix is **`".bmmap"`**.


Summary
----------

Generating and utilizing the above defined lightweight JSON-Mmap table along side with a
JSON or binary JSON document may permit a user to rapidly locate, read, overwrite or extend
user-interested data records without needing to reading/parsing/writing the entire document,
resulting in signficant file IO performance improvement. It is particularly beneficial for
read-only access of a hierachical JSON/binary JSON document and one can directly utilize
conventional `mmap` based disk access to rapidly retrieve the data using the information
encoded in the JSON-Mmap table. We anticipate that enabling `mmap` access to JSON and binary
JSON data files can dramaticaly accelerate reading/writing efficiency in speed critical
applications without adding major overhead.
