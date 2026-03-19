# Technical Specification: Hybrid Linked-Data Document (HLD) v1.0

This specification describes the **Hybrid Linked-Data Document (HLD)** format.

HLD is geared towards being a **database file-format** for **Linked-Data** that supports **CRUD** (**Create Read Update Delete**) operations.

HLD can be compared to the HDT (Header, Dictionary, Triples) format.
While a format such as HDT (Header, Dictionary, Triples) is excellent for read opertations, performs poorly for write and update operation.
HLD works well for write and update operations.

HLD can also be compared to the JSON-LD format.
JSON-LD is geared towards programmer-legibiilty, where HLD is geared towards machine-legibililty and performance.

---

## 1. File Structure Overview

The HLD format uses a **Paged Binary Layout**.

An HLD file is divided into 4KB segments (Pages) to allow for efficient disk I/O and random access.
(Most extant operating-system seem to have Pages that are 4KB sized. Which is why _4KB_ was chosen.)

| Segment             | Purpose                                                                                                            |
| :------------------ | :----------------------------------------------------------------------------------------------------------------- |
| **Magic Header**    | Used to determine if file is actually is an HLD file. "HLD/1.0\n" (48 4C 44 2F 31 2E 30 0A)                        |
| **Dictionary**      | Used to reduce the size of the data in the _Data Pages_. A mapping of URIs and common Keys to Short-IDs (Varints). |
| **Data Pages**      | Document-centric blocks stored in **CBOR** (Concise Binary Object Representation).                                 |
| **Index (Trailer)** | A B-Tree or Hash Index for lookup by Document ID or Subject.                                                       |
