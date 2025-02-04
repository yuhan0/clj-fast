[![Clojars Project](https://img.shields.io/clojars/v/bsless/clj-fast.svg)](https://clojars.org/bsless/clj-fast)
[![cljdoc badge](https://cljdoc.org/badge/bsless/clj-fast)](https://cljdoc.org/d/bsless/clj-fast/CURRENT)
[![CircleCI](https://circleci.com/gh/bsless/clj-fast/tree/master.svg?style=shield)](https://circleci.com/gh/bsless/clj-fast/tree/master)

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [clj-fast](#clj-fast)
    - [Purpose](#purpose)
    - [Latest Version](#latest-version)
    - [Results](#results)
    - [Usage](#usage)
        - [Requirements](#requirements)
        - [Functions and Macros](#functions-and-macros)
            - [Fast(er) Functions](#faster-functions)
            - [Inline Macros](#inline-macros)
                - [Notes](#notes)
                - [Additions](#additions)
            - [Bypass dynamic dispatch with type hints](#bypass-dynamic-dispatch-with-type-hints)
            - [Collections](#collections)
                - [HashMap](#hashmap)
                - [ConcurrentHashMap](#concurrenthashmap)
            - [Lenses](#lenses)
    - [Rewriting Core Functions And Macros](#rewriting-core-functions-and-macros)
        - [Usage](#usage-1)
    - [Related Projects](#related-projects)
        - [Structural](#structural)
        - [Stringer](#stringer)
    - [License](#license)
    - [Credit](#credit)

<!-- markdown-toc end -->
# clj-fast

Library for playing around with low level Clojure code for performance
reasons given some assumptions. Inspired by [Naked Performance (with
Clojure) – Tommi Reiman](https://www.youtube.com/watch?v=3SSHjKT3ZmA).

Some of the code is based on implementations in metosin's projects. Credit in code.

## Purpose

This repo serves a dual purpose:

- Providing faster implementations of Clojure's core functions as
  macros.
- Reference guide on the performance characteristics of different ways
  of using Clojure's data structures.

What makes it possible?

Plenty of Clojure's core functions are implemented to be generic (good)
and to accept a variable number of arguments (also very good). The
problem is that we pay for this in performance. Wherever we iterate over
a sequence of input arguments or dispatch on the class, we lose on
performance, especially when iterating on arguments and calling `next`,
`more` or `rest` repeatedly.

Plenty of these behaviors are just forms of flow-control, and like `and`
and `or`, other forms of flow control can too be statically analyzed,
under certain constraints, and replaced by faster code.

## Latest Version

[![Clojars Project](http://clojars.org/bsless/clj-fast/latest-version.svg)](http://clojars.org/bsless/clj-fast)

## Results

See [results.md](doc/results.md) for experiments' detailed benchmark results.

## Usage

### Requirements

Add in your `project.clj`:

```clojure
[bsless/clj-fast "0.0.9"]
```

WARNING: Due to a bug in leiningen, versions built prior to `0.0.9` will pull in extra dependencies. Make sure to upgrade!

### Functions and Macros

#### Fast(er) Functions

```clojure
(require '[clj-fast.core :as fast])
```

- `entry-at`: used like `find` but doesn't dispatch and has inline
  definition. Works for `IPersistentMap`.
- `val-at`: used like `get` but doesn't dispatch and has inline
  definition. Works for `IPersistentMap`.
- `fast-assoc`: Used like `assoc` but doesn't take variable key-values,
  only one pair and has inline definition. Works on `Associative`.
- `fast-map-merge`: Slightly faster version for `merge`, takes only 2
  maps.
- `rmerge!`: merges a map into a transient map.


#### Inline Macros

```clojure
(require '[clj-fast.inline :as inline])
```

Like regular core functions but sequence arguments must be written
explicitly for static analysis or `def`ed in advance (i.e. `resolve`-able).

Examples:

```clojure
(def ks [:a :b])

(inline/assoc m :a 1 :b 2)

(inline/dissoc m :a :b)

(inline/fast-assoc m :a 1 :b 2)

(inline/get-in m ks)

(inline/get-in m [:c :d])

(inline/get-some-in m [:c :d])

(inline/assoc-in m [:c :d] foo)

(inline/assoc-in m [:c :d] foo [:c :b] bar)

(inline/update-in m [:c :d] inc)

(inline/select-keys m [:a :b :c])

(inline/merge m1 m2 m3)

(def assoc* (inline/memoize-c 3 assoc))
```

##### Notes

- Merge analysis unrolls inline maps as well.
- Warning: additional arities of assoc-in will cause code reordering.
  Beware of side effects.

##### Additions

- `fast-assoc`: inlines in the same manner of `assoc` but uses
  `clj-fast.core/fast-assoc` instead.
- `fast-map-merge`: inlines in the same manner of `merge` but uses
  `clj-fast.core/fast-map-merge` instead (Metosin).
- `fast-select-keys`: like `select-keys`, but faster and dirtier, adds
  nil entries to the results map.
- `get-some-in`: Like `get-in` at the expense of working only on callables
  (objects implementing `clojure.lang.IFn`).
- `find-some-in`: like `get-some-in` but returns a map-entry in the end,
  like `find`.
- `memoize*` & `memoize-c*`: Alternative implementations for memoization
  using a nested Clojure hash map and a nested Java concurrent hash map
  respectively. Fall back to `core/memoize` for large arities. Due to
  the cost of hashing objects in Clojure, it's recommended to use
  `memoize-c*` for most use cases.

#### Bypass dynamic dispatch with type hints

The `get` and `nth` macros operate similarly to their respective
functions with one notable difference: When provided with an appropriate
type hint, they will dispatch to the underlying method at compile time
instead of run time.

```clojure
(def arr (long-array [1 2 3]))
(nth ^longs arr 0)
(def m (doto (java.util.HashMap.) (.put :a 1)))
(get ^Map m :a)
```

#### Collections

##### HashMap

```clojure
(require '[clj-fast.collections.hash-map :as hm])
```

- `->hashmap`: wraps `HashMap`'s constructor.
- `get`: wraps method call for `HashMap`'s `get`. Has inline definition.
- `put`: wraps method call for `HashMap`'s `put`. Has inline definition.

##### ConcurrentHashMap

```clojure
(require '[clj-fast.collections.concurrent-map :as chm])
```

- `->concurrent-hash-map`: constructor.
- `concurrent-map?`: instance check.
- `put!?`: `putIfAbsent`.
- `get`
- `get?`: get if is a concurrent hash map.
- `get-in?`: like clojure core's get-in but for nested concurrent hash maps.
- `put-in!`: like clojure core's assoc-in but for nested concurrent hash maps.

#### Lenses

```clojure
(require '[clj-fast.lens :as lens])
```

In typed functional programming, lenses are a generic way of getting
and setting nested data structures (records).

In this context, the `lens` namespace implements the basic code structure
underlying Clojure's `get-in`, `some->`, `assoc-in` and `update-in`.
They can be used in macros to expand to real code provided an appropriate
1-depth `get` and/or `put` transformer, which takes arguments and returns
an expression.

For example, the `get-some` lens is used to define `inline/get-some-in`:

```clojure
(defmacro get-some-in
  [m ks]
  (lens/get-some (fn [m k] `(~m ~k)) m ks))
```

Similarly, for `assoc-in`:

```clojure
(defmacro assoc-in
  [m ks v]
  (lens/put
   (fn [m k v] `(c/assoc ~m ~k ~v))
   (fn [m k] `(c/get ~m ~k))
   m
   (u/simple-seq ks)
   v))
```

So be careful, these are not functional programming lenses, but
metaprogramming lenses used for code generation.

## Rewriting Core Functions And Macros

The namespace `clj-fast.clojure.core` contains drop-in replacement
functions and macros for Clojure's core.

It opportunistically replaces functions by their inlined
implementations. It also includes binding macros (let, fn, loop, defn)
which will use inlining versions of `get` and `nth` when possible. (i.e.
when type-hinted).

### Usage

```clojure
(ns com.my.app
  (:refer-clojure
   :exclude
   [get nth assoc get-in merge assoc-in update-in select-keys memoize destructure let fn loop defn defn-])
  (:require
   [clojure.core :as c]
   [clj-fast.clojure.core :refer [get nth assoc get-in merge assoc-in update-in select-keys memoize destructure let fn loop defn defn-]]))
```

## Related Projects

### Structural

(Structural)[https://github.com/joinr/structural] is a small library by
joinr (Tom) which provides more efficient destructuring macros with type
hints.

### Stringer

(Stringer)[https://github.com/kumarshantanu/stringer] is a library by
Shantanu Kumar for fast string operations. Of interest are the
capabilities it provides in faster string building and formatting, also
by "unrolling" the building operations where statically possible.

## License

Copyright © 2019-2020 ben.sless@gmail.com

Copyright © Rich Hickey for the implementation in `clj-fast.clojure.core`.

This program and the accompanying materials are made available under the
terms of the Eclipse Public License 2.0 which is available at
http://www.eclipse.org/legal/epl-2.0.

This Source Code may also be made available under the following Secondary
Licenses when the conditions for such availability set forth in the Eclipse
Public License, v. 2.0 are satisfied: GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or (at your
option) any later version, with the GNU Classpath Exception which is available
at https://www.gnu.org/software/classpath/license.html.

## Credit

Credit to Metosin wherever noted in the code.

Rich Hickey for clojure.core ns.
