# FileSystemHandle Unique ID

## Authors:

* Austin Sullivan (asully@chromium.org)

## Participate

* [Issue tracker](https://github.com/whatwg/fs/issues)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals & Use Cases](#goals--use-cases)
- [Non-goals](#non-goals)
- [Use Cases](#use-cases)
  - [O(1) access to handles in IndexedDB](#o1-access-to-handles-in-indexeddb)
    - [Application-level resource identification](#application-level-resource-identification)
    - [Application-level locking using WebLocks](#application-level-locking-using-weblocks)
    - [Storing a representation of a handle in the database of your choice](#storing-a-representation-of-a-handle-in-the-database-of-your-choice)
- [Why Hashing Isn’t Enough](#why-hashing-isnt-enough)
- [Alternatives Considered](#alternatives-considered)
    - [Maintain status quo: O(n) lookups of a `FileSystemHandle` in IndexedDB](#maintain-status-quo-on-lookups-of-a-filesystemhandle-in-indexeddb)
    - [Expose the full file path](#expose-the-full-file-path)
    - [Expose the same ID to all sites](#expose-the-same-id-to-all-sites)
    - [Expose a method to convert an ID to its corresponding `FileSystemHandle`](#expose-a-method-to-convert-an-id-to-its-corresponding-filesystemhandle)
    - [Tie the salt to the lifetime of the current browsing session](#tie-the-salt-to-the-lifetime-of-the-current-browsing-session)
    - [Expose a truly persistent ID which is stable even after clearing site data](#expose-a-truly-persistent-id-which-is-stable-even-after-clearing-site-data)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Currently, `FileSystemHandle` objects can be opaquely serialized by the browser
to be stored as values in IndexedDB. But there is no way for a site to generate
a string from script which is guaranteed to be uniquely identifying for the file
referenced by the `FileSystemHandle`.

Developers have complained that building rich experiences on top of the File
System Access API is challenging due to the inability to uniquely identify (and
index on) handles.

## Goals & Use Cases

* Support O(1) access to a FileSystemHandle stored in IndexedDB
* Two colluding sites cannot “join” a user by comparing IDs server-side
* IDs should be stable across browsing sessions, making it suitable to use as a
  key for storing the corresponding FileSystemHandle in IndexedDB
* IDs should not be stable after clearing browsing data. In other words, this
  should not be a truly persistent, unclearable identifier
* IDs should not be stable across normal browsing mode and private browsing
  modes
* No information should be gleaned from the ID. It is completely opaque to the
  site
* Avoid surprises by having the same semantics as `isSameEntry()`, namely:
  `await a.isSameEntry(b) === (await a.getUniqueId() === await b.getUniqueId())`

## Non-goals

* Support obtaining a `FileSystemHandle` from its ID

## Use Cases

### O(1) access to handles in IndexedDB

An application wants to quickly tell whether it’s already seen a file which was
recently selected from the file picker. Currently, storing handles in IndexedDB
requires creating an arbitrary key mapping to a list of handles. Identifying
whether a given handle is already stored in IndexedDB requires iterating through
this list and calling the `isSameEntry()` method on each handle - an O(n)
operation which is unacceptable given that sites may have access to an unbounded
number of handles. A `FileSystemHandle`’s unique ID can be used as a key for
IndexedDB, allowing for constant-time access.

Before:

```javascript
// IndexedDB structure
// { "handles": [handle1, handle2, … ] }
const key = await handle.getUniqueId();
var handles = await get("handles");
handles.push(handle);
await set("handles", handles);
```

After:

```javascript
// IndexedDB structure
// { 
//   "abc123": handle1,
//   "def456": handle2,
//    …
// }
const key = await handle.getUniqueId();
await set(key, handle);
```

#### Application-level resource identification

A web IDE app wants to display a list of “recently opened” directories, with
buttons to jump back into editing. The ID can be used as a pointer to associate
various page elements and Javascript objects with the corresponding
`FileSystemHandle`.

#### Application-level locking using WebLocks

A document-editing app wants to prevent concurrent modification to the same
file. The File System Access API has a built-in API-level mechanism to
exclusively lock files with an open `SyncAccessHandle`, but there is no way to
exclusively lock a `FileSystemHandle` being written to with a
`FileSystemWritableFileStream`, which is the only way to write to files which
live outside of the origin-private file system. The WebLocks API takes a string
parameter as a key, which can be used to lock a file handle at the application
level. 

```javascript
const key = await handle.getUniqueId();
await navigator.locks.request(key, async lock => {
  const writable = await handle.createWritable();
  // Write without fear of conflicts.
});
```

#### Storing a representation of a handle in the database of your choice

An app wants to store a unique identifier to a file client-side, such as
[in a locally-running SQLite instance](https://sql.js.org/#/), or server-side,
such as [in a Redux Store](https://github.com/WICG/file-system-access/issues/368). 

Currently, `FileSystemHandle` objects themselves can only be stored in
IndexedDB, since they are
[serializable](https://html.spec.whatwg.org/multipage/structured-data.html#serializable)
only by the structured cloning algorithm. If the app wants to store other data
associated with this handle, it needs to either reside in IndexedDB alongside
the handle or use complicated (and likely fragile) workarounds to associate this
other data with the file.

With this new method, the string ID can effectively be used as a pointer (from
the database of the app’s choice) to the handle in IndexedDB.

## Why Hashing Isn’t Enough

A string key can be generated from script by hashing the currently exposed
fields of the `FileSystemHandle`, such as the name, last-modified time, and file
contents. Sites can be reasonably sure that two handles are the same file by
hashing all of these fields (though even this is insufficient to de-duplicate a
handle from its copy in another directory). However, this does not provide much
value, since this point-in-time comparison is already provided by the
`isSameEntry()` method. For the hash to be useful, it must be (1) stable, even
as the file changes, (2) fast to compute, and (3) unique.

It is not possible to create a hash with these properties using the existing
API surface.

1. The underlying file can be modified, possibly without the site’s awareness.
   Using the file contents or last-modified time will lead to a possibly
   unstable hash.
2. Hashing the file contents can be prohibitively expensive.
3. The remaining hashable fields on the `FileSystemHandle` do not provide enough
   information to be able to distinguish between two files of the same name in
   different directories, for example.

## Alternatives Considered


#### Maintain status quo: O(n) lookups of a `FileSystemHandle` in IndexedDB

This is not feasible for sites which may have access to an unbounded number of
handles and need to determine quickly whether they’ve already stored a given
handle.


#### Expose the full file path

The full file path often contains PII, such as a username, that we do not wish
to expose to the web. We could gate this method behind a permission prompt, but
it is not reasonable to expect a user to understand the implications of granting
access.


#### Expose the same ID to all sites

This would allow de-duplication of files across sites. While this may be useful
for well-behaving sites, this generally goes against the per-origin defaults of
the web and has the potential abuse from misbehaving sites (i.e. two colluding
sites can "join" a user by comparing IDs server-side).


#### Expose a method to convert an ID to its corresponding `FileSystemHandle`

This would introduce yet another entry point to the API, complicating future
security and privacy discussions. There is little added value to being able to
obtain a handle from its ID directly, since this feature enables a site to use
the ID to access the handle quickly from IndexedDB anyways.


#### Tie the salt to the lifetime of the current browsing session

This would invalidate all prior IDs if the page is refreshed, making them
significantly less useful as keys to any storage which lasts beyond the current
browsing session (such as IndexedDB).


#### Expose a truly persistent ID which is stable even after clearing site data

This would allow sites to de-duplicate handles by persisting the ID to their
servers. However, this would make the ID effectively an unclearable cookie.
Supporting this use case is not worth the privacy cost, especially since the
most common case will likely involve storing the ID in IndexedDB, which will be
among the site data cleared.

## Stakeholder Feedback / Opposition

*   Developers: Positive
    *   https://github.com/WICG/file-system-access/issues/295

## References & acknowledgements

Many thanks for valuable feedback and advice from:

Ayu Ishii, Joshua Bell, Thomas Steiner
