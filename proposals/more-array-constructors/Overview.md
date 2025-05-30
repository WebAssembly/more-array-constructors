# More Array Constructors

## Summary

Introduce three new array-construction instructions:

| Instruction                   | Type                                      | Informal Description |
|-------------------------------|-------------------------------------------|----------------------|
| <pre><code>array.new_array dst src</code><br/></pre>     | <pre><code>[(ref null src) i32 i32] -> [(ref dst)]</code><br/></pre> | Create an array of type `dst` from a subslice of the elements of an array of type `src`. Element subslice is given as `(start, length)`. Validation requires `elements(dst) <: elements(src)`. Traps on out-of-bounds. |
| <pre><code>array.new_memory dst src</code><br/></pre>    | <pre><code>[i32 i32] -> [(ref dst)]</code><br/></pre>                | Create an array of type `dst` from a subslice of the memory `src`. Memory subslice is given as `(start, length)` where `length` is in units of elements, not bytes. Validation requires that the memory `src` exists and that elements of `dst` have a defined bit width. Traps on out-of-bounds. |
| <pre><code>array.new_table dst src</code><br/></pre>     | <pre><code>[i32 i32] => [(ref dst)]</code><br/></pre>                | Create an array of type `dst` from a subslice of the table `src`. Table subslice is given as `(start, length)`. Validation requires that the table `src` exists and that `elements(dst) <: elements(src)`. Traps on out-of-bounds. |

## Motivation

The methods for constructing immutable arrays is rather limited at the
moment. In particular, it is impossible to create an immutable array with both
dynamic (defined at runtime) length as well as dynamic and heterogeneous
contents.

Consider the current set of instructions for allocating arrays:

| Instruction         | Length  | Contents               |
|---------------------|---------|------------------------|
| `array.new`         | dynamic | dynamic, homogeneous   |
| `array.new_default` | dynamic | static, homogeneous    |
| `array.new_fixed`   | static  | dynamic, heterogeneous |
| `array.new_data`    | dynamic | static[^semi-static], heterogeneous  |
| `array.new_data`    | dynamic | static[^semi-static], heterogeneous  |

[^semi-static]: For `array.new_data` and `array.new_elem`, the contents are
    arguably not *fully* static, as they may be any subslice of a pre-existing
    data or element segment. The important bit is that, because new segments
    cannot be constructed at runtime, the instruction does not allow for fully
    dynamic, arbitrary contents defined at runtime.

Mutable arrays can always have their contents updated after construction. For
immutable arrays, we do not have that luxury, and creating an immutable array of
dynamic length and dynamic, heterogeneous contents requires the following,
missing row:

| Instruction         | Length  | Contents               |
|---------------------|---------|------------------------|
| ???                 | dynamic | dynamic, heterogeneous |

For this hypothetical instruction, where should the dynamic contents come from?
Because they have dynamic length, they cannot come from the value stack; that
would break validation. A Wasm program might want any one of three options:

1. **From another array:** The Wasm program might want to copy an existing
   array's contents, or a subslice of those contents, into a new array.
2. **From a memory:** The Wasm program might want to copy a slice of memory into
   a new array (which must be an array whose element type has a defined bit
   width, i.e. not an array of references).
3. **From a table:** The Wasm program might want to copy a slice of table
   elements into a new array (which must be an array whose element type is a
   reference type).

From an expressivity perspective, any single one of those options can unlock the
construction of dynamic, heterogeneous, immutable arrays and the other
operations could be built on top of it. If you had an instruction for creating
an immutable array from a subslice of memory, for example, you could implement
creating an immutable array from a subslice of another array by first copying
the source array's subslice to memory and then using the
array-from-memory-subslice instruction to create the new array. That said, while
it is possible to implement these operations in terms of one another, having an
instruction for each variant should be much easier for engines to optimize.

### Why Should We Care About Immutable Arrays?

* **Zero-copy, shared-nothing linking:** [Shared-nothing
  linking](https://github.com/WebAssembly/component-model/blob/31256b8d0991c1d84edc8f889938103d3d62faef/design/mvp/Linking.md?plain=1#L17-L22)
  requires that components are isolated from one another and that they *only*
  communicate with each other via explicit imports and exports. They cannot, for
  example, share some state (like a memory, for example) that would allow a
  malicious component to smash and overwrite another component's data in that
  shared state. When doing shared-nothing linking, we can pass immutable arrays
  at the ABI boundary without copying. This is safe because we know that even if
  the sender holds onto a reference to the array, it cannot mutate that array
  after it has been shared with the receiver, which could otherwise lead to
  sandboxing bugs, data smashing, and side-channels that break the
  shared-nothing paradigm.

* **More opportunities for compiler optimizations:** Because immutable objects
  do not change, the compiler is free to eliminate redundant loads, replacing
  their uses with uses of the previously-loaded value (essentially GVN'ing
  loads) without first performing any complex and potentially expensive alias
  analysis. This leads to better code generation and faster Wasm execution while
  maintaining compiler simplicity and fast compile times.

* **More opportunities for specialized garbage collection:** Immutable objects
  cannot create cycles, which creates opportunities for garbage collectors to
  special case these objects. A reference counting collector that has a periodic
  cycle collection phase need not consider an immutable sub-graph in its cycle
  collector. Alternatively, a collector that generally uses tracing might decide
  to use reference counting for immutable sub-graphs of objects once they
  graduate from the nursery to the tenured heap, combining some of the
  incremental benefits of reference counting with the throughput and
  completeness of tracing. Furthermore, certain kinds of GC barriers can be
  completely omitted for immutable objects: an immutable object can never point
  to another object that is younger than itself so in a generational collector,
  for example, an immutable object need not record its edges in the remembered
  set. These kinds of specializations lead to less work for the collector and,
  ultimately, better mutator throughput and shorter GC pauses.

In general, however, we only get to leverage these benefits for immutable arrays
in practice if Wasm can create arbitrary immutable arrays, and that requires
something like the instructions proposed above. These proposed instructions are
pretty trivial to implement but provide out-sized benefit for Wasm programs;
that is, these instructions yield a lot of relative bang for a little buck. The
language already has immutable arrays and engines must already support them, so
we might as well actually make them useful.
