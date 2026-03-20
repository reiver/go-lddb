# Technical Specification: Hybrid Linked-Data Document (HLD) v1.0

This specification describes the **Hybrid Linked-Data Document (HLD)** format.

**HLD** is geared towards being a **database file-format** for **Linked-Data** that supports **CRUD** (**Create Read Update Delete**) operations.

**HLD** can be compared to the **HDT** (**Header, Dictionary, Triples**) format.
While a format such as **HDT** (**Header, Dictionary, Triples**) is excellent for read opertations, it (**HDT**)performs poorly for write and update operation.
**HLD** (**Hybrid Linked-Data Document**) works well for write and update operations.

**HLD** can also be compared to the **JSON-LD** format.
**JSON-LD** is geared towards programmer-legibiilty, where **HLD** is geared towards machine-legibililty and performance.

---

## 1. File Structure Overview

The **HLD** format uses a **Paged Binary Layout**.

An **HLD** file is divided into 4KB segments (Pages) to allow for efficient disk I/O and random access.
(Most extant operating-system seem to have Pages that are 4KB sized. Which is why _4KB_ was chosen.)

| Segment             | Purpose                                                                                                            |
| :------------------ | :----------------------------------------------------------------------------------------------------------------- |
| **Magic Header**    | Used to determine if file is actually is an HLD file. "HLD/1.0\n" (48 4C 44 2F 31 2E 30 0A)                        |
| **Dictionary**      | Used to reduce the size of the data in the _Data Pages_. A mapping of URIs and common Keys to Short-IDs (Varints). |
| **Data Pages**      | Document-centric blocks stored in **CBOR** (Concise Binary Object Representation).                                 |
| **Index (Trailer)** | A B-Tree or Hash Index for lookup by Document ID or Subject.                                                       |

---

## 2. The Dictionary Layer (The "HDT-Lite" Secret)

**Dictionaries** are sometimes used to compress data.
The **HLD** format (also) uses a **dictionary** to compress data.

Here we are using **"dictionary"** in the computing sense of the word.
I.e., a **dictionary** is a mapping from one thing to another.
For the purposes of this specification, we are mapping strings (such as URLs/URIs/IRIs, keys, etc) to variable-length integers.

The basic idea is to replace frequently occuring strings with a short integer.

For example, maybe replacing the string:

```
"https://www.w3.org/ns/activitystreams#Person"
```

With:

```
3
```

We can look at this same example from a **byte** point of view.
This is the string `"https://www.w3.org/ns/activitystreams#Person"`:

```
74 74 70 73 3a 2f 2f 77 77 77 2e 77 33 2e 6f 72 67 2f 6e 73 2f 61 63 74 69 76 69 74 79 73 74 72 65 61 6d 73 23 50 65 72 73 6f 6e
```

And this is the short integer that could replace the string:

```
03
```

The short integer is interpreted as an _index_ into a **dictionary** (look-up table).

The **dictionary** (look-up table) could be:

```
1 → `"@id"`
2 → `"@type"`
3 → `"https://www.w3.org/ns/activitystreams"`
4 → `"http://schema.org"`
5 → `"http://purl.org/dc/elements/1.1/"
6 → `"https://www.w3.org/ns/activitystreams#Person"`
7 → `"https://www.w3.org/ns/activitystreams#name"`
...
```

To see this from the point of view of a JSON-LD document, the following JSON-LD

```json
{"@type": "Person", "name": "Alice"}
```

Might be replaced with:

```
[0x02, 0x06, 0x07, "Alice"]
```

Where `0x02` maps to `"@type"`, `0x06` maps to `"https://www.w3.org/ns/activitystreams#Person"`, and `0x07` maps to `"https://www.w3.org/ns/activitystreams#Person"`.

### 2.1. Dictionary Sections

The dictionary will have 2 sections:

| Section Type       | Section Description                                                           |
|--------------------|-------------------------------------------------------------------------------|
| Static Dictionary  | Part of the specification. Values not stored in file. For very common strings. Drawn from common vocabularies (activitypub/activitystreams, schema.org, rdf, rdfs). |
| Dynamic Dictionary | Determined from the data. (Ex: common URLs/URIs/IRIs, other common strings.)  |

---

## 3. Data Page Encoding
Each **Data Page** contains a cluster of **JSON-LD documents**.

**JSON-LD documents** are encoded as CBOR.

### 3.1. Fragments

The **format** is: CBOR-encoded fragments.

A **JSON-LD document** (encoded as CBOR) _could_ be larger than a single page.
To deal with this, we will say that a single Page contains CBOR-encoded **fragment**.
(Of course, if the CBOR encoded JSON-LD document is small enough, there will be only 1 fragment that will be the whole document.)

Alternatively, if many JSON-LD documents are small enough, many JSON-LD document could fit in a single page.

### 3.2. Referential Integrity

Instead of the CBOR objects storing the the URLs/URIs/IRIs used as IDs (identifiers) for the JSON-LD objects, the encoded CBOR will store **relative pointer offsets**.

For example, a value of `-50` could mean, see the document that starts _50 bytes back_ in the file from here.
Or, for example, a value of `100` could mean, see the document that starts _100 bytes forward_ in the file from here.

### 3.3. Graph Flattening

The JSON-LD documents are "flattened" before storing them as CBOR.
So that there aren't any embedded objects.
The embedded objects get their own CBOR object (and the thing that _had_ it embedded in it points to it).

The reasons for this:

* **sequential access**: a parser can, for example read an "Address" into memory, then the "Person" (that was embedding it), and simply point the Person's address field to the memory location where the Address already sits
* **stack overflow prevention**: with this (and unlike JSON) there is no deep recursion; you are simply iterating through a list of entries
* **partial reads**: if, for example, you only need the "Address", you don't have to parse the entire "Person" object to find it

Here is an example.
Consider this example JSON-LD document:

```json
{
  "@id": "https://example.com/~joeblow",
  "name": "Joe Blow",
  "address": {
    "locality": "London",
    "postal-code": "SW1A 2AA"
}
```

After flattening there would be 2 JSON-LD nodes:

```json
{
  "locality": "London",
  "postal-code": "SW1A 2AA"
}
```

```json
{
  "@id": "https://example.com/~joeblow",
  "name": "Joe Blow",
  "address": [POINTER TO THE PREVIOUS ADDRESS RECORD]
}
```

---

## 4. Indexing Model

---

## 5. Implementation Requirements

###  5.1. Alignment

To make this performant, there are some quirks of modern computer and CPU architecture that we want to optimize for.

What is considered _modern CPUs_ (at the time of writing) (such as x86_64 or ARM64) do not actually read data from memory one byte at a time.
They fetch data from memory in "chunks" (which tend to be 8 bytes or 64 bits at a time).
If a 64-bit integer starts at an address that isn't a multiple of 8, the CPU has to perform two memory fetches and a bit-shift to "stitch" the number together.

Therefore, in this format, all blocks must be 8-byte aligned (so that we can optimize for current modern CPU architectures).

### 5.2. Concurrency

The format must support **single-write / multiple-readers** (**SWMR**).
Perhaps using _file locking_.

### 5.3. Zero-Copy

Copying data over and over again can negatively affect performace.

A parser should _map_ the file into memory (`mmap`) and return pointers to the strings in the Dictionary rather than copying them into new memory strings.

---

