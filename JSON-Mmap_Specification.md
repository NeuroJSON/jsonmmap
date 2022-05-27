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
- [Grammar](#grammar)
    * [Path string](#path-string)
    * [Locator vector](#locator-vector)
    * [Metadata keys and values](#metadata-keys-and-values)
    * [Storage of the JSON-Mmap table](#storage-of-the-json-mmap-table)
- [Sample utilities of JSON-Mmap](#sample-utilities-of-json-mmap)
    * [Fast data-record disk reading](#fast-data-record-disk-reading)
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

```
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
    - a string starting with ASCII letter `$` (`0x24` in hex or ) is referred to as a **path**,
      representing a JSON-Path like reference, pointing towards a specific data record to the
      mapped JSON/binary JSON file, and
    - any string that does not start with `$` is referred to as a **metadata key**, to store optional
      auxillary information related to the JSON-Mmap.
- when the first string is a **path**, the second element must be an array of integers, hereinafter
  referred to as the **locator**, containing various information regarding the byte-location of the
  serialized data record on the disk or memory buffer
- when the frist string is a **metadata key**, the second element denotes the value of the metadata,
  some recommended metadata key value formats are listed below.

### Path string

The format of the path string is inspired and largely similar to the syntax of
[JSON-Path](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html), where
the string must follow the below format

|      Notation     |                                                   Meaning                                                   |
|-------------------|-------------------------------------------------------------------------------------------------------------|
|`$`                | the root object, must be the first letter of all paths                                                      |
|`$[i]`             | i=0,1,2,..., denoting the (i+1)th root-level object in an concatenated JSON (CJSON) or binary JSON document |
|`.`                | child of the object on the left                                                                             |
|`[i]`              | i=0,1,2,..., denoting the (i+1)th child of an array on the left                                             |
|`.key` or `['key']`| a named child with a string-typed name `key`; using `['key']` is required when `key` has `.` ,`[` or `]`    |

Note: JSON-Mmap path specifier does not support additional annotations in JSONPath, such as `..` or `@` operator

For example, for the below JSON (or JSON-like) object
```
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
[<offset>, <length>, <white-space-byte-length-after>, <white-space-byte-length-before> ...]
```
where

- (recommended) the first number `<offset>`, if present, must be an integer, denoting the byte-offset of the first
  significant character (i.e. must not be a white-space or `no-op` marker) of the referenced value
  from the begining of the referenced document, 
- (recommended) the second number `<length>`, if present, must be an intger, denoting the end-to-end byte-length
  of the referenced value between the first significant character and the last significant character
  (inclusive)
- (optional) the third number, `<white-space-byte-length-after>`, if present, must be an integer, denoting the
  byte-lengt of all whitespaces (non-significant character) after the last signficiant character and
  before the separator mark `,` with the following objects, or the next significant character of the parent object.
- (optional) the third number, `<white-space-byte-length-before>`, if present, must be an integer, denoting the
  byte-lengt of all whitespaces (non-significant character) before the first signficiant character and
  after the separator mark `,` with the preceding objects, or a significant character of the enclosing parent object.
- additional elements in the locator vector are reserved for future extension of this specification

## Metadata keys and values


### Storage of the JSON-Mmap table


Suggested use of JSON-Mmap
------------------------

### Fast data-record disk reading


### On-disk value replacement with equal or less bytes


### On-disk value replacement with more bytes


Recommended File Specifiers
------------------------------

For the text-based JData file, the recommended file suffix is **`".jmmap"`**; for 
the binary JData file, the recommended file suffix is **`".bmmap"`**.


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
