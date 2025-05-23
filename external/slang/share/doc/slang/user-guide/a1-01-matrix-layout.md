---
layout: user-guide
---

Handling Matrix Layout Differences on Different Platforms
=========================================================

The differences between default matrix layout or storage conventions between GLSL (OpenGL/Vulkan) and HLSL has been an issue that frequently causes confusion among developers. When writing applications that work on different targets, one important goal that developers frequently seek is to make it possible to pass the same matrix generated by host code to the same shader code, regardless of what graphics API is being used (e.g. Vulkan, OpenGL or Direct3D). As a solution to shader cross-compilation, Slang provides necessary tools for developers navigate around the differences between GLSL and HLSL targets.

A high level summary:

* Default matrix **layout** in memory for Slang is `row-major`. 
  * Except when running the compiler through the `slangc` tool, in which case the default is `col-major`. This default is for *legacy* reasons and may change in the future.
* Row-major layout is the only *portable* layout to use across targets (with significant caveats for non 4x4 matrices)
* Use `setMatrixLayoutMode`/`spSetMatrixLayoutMode`/`createSession` to set the default  
* Use `-matrix-layout-row-major` or `-matrix-layout-column-major` for the command line 
  * or via `spProcessCommandLineArguments`/`processCommandLineArguments`
* Depending on your host maths library, matrix sizes and targets, it may be necessary to convert matrices at host/kernel boundary  
  
On the portability issue, some targets *ignore* the matrix layout mode, notably CUDA and CPU/C++. For this reason for the widest breadth of targets it is recommended to use *row-major* matrix layout.

Two conventions of matrix transform math
----------------------------------------

Depending on the platform a developer is used to, a matrix-vector transform can be expressed as either `v*m` (`mul(v, m)` in HLSL), or `m*v` (`mul(m,v)` in HLSL). This convention, together with the matrix layout (column-major or row-major), determines how a matrix should be filled out in host code. 

In HLSL/Slang the order of vector and matrix parameters to `mul` determine how the *vector* is interpreted. This interpretation is required because a vector does not in as of it's self differentiate between being a row or a column.

* `mul(v, m)` - v is interpreted as a row vector.
* `mul(m, v)` - v is interpreted as a column vector.

Through this mechanism a developer is able to write transforms in their preferred style. 

These two styles are not directly interchangeable - for a given `v` and `m` then generally `mul(v, m) != mul(m, v)`. For that the matrix needs to be transposed so

* `mul(v, m) == mul(transpose(m), v)`
* `mul(m, v) == mul(v, transpose(m))`

This behavior is *independent* of how a matrix layout in memory. Host code needs to be aware of how a shader code will interpret a matrix stored in memory, it's layout, as well as the vector interpretation convention used in shader code (ie `mul(v,m)` or `mul(m, v)`).

[Matrix layout](https://en.wikipedia.org/wiki/Row-_and_column-major_order) can be either `row-major` or `column-major`. The difference just determines which elements are contiguous in memory. `Row-major` means the rows elements are contiguous. `Column-major` means the column elements are contiguous.

Another way to think about this difference is in terms of where translation terms should be placed in memory when filling a typical 4x4 transform matrix. When transforming a row vector (ie `mul(v, m)`) with a `row-major` matrix layout, translation will be at `m + 12, 13, 14`. For a `column-major` matrix layout, translation will be at `m + 3, 7, 11`.

Note it is a *HLSL*/*Slang* convention that the parameter ordering of `mul(v, m)` means v is a *row* vector. A host maths library *could* have a transform function `SomeLib::transform(v, m)` such that `v` is a interpreted as *column* vector. For simplicitys sake the remainder of this discussion assumes that the `mul(v, m)` in equivalent in host code follows the interpretation that `v` is *row* vector.

Discussion
----------

There are four variables in play here:

* Host vector interpretation (row or column) - and therefore effective transform order (column) `m * v` or (row) `v * m`
* Host matrix memory layout
* Shader vector interpretation (as determined via `mul(v, m)` or `mul(m, v)` )
* Shader matrix memory layout 

Since each item can be either `row` or `column` there are 16 possible combinations. For simplicity let's reduce the variable space by making some assumptions.

1) The same vector convention will be used in host code as in shader code. 
2) The host maths matrix layout is the same as the kernel.

If we accept 1, then we can ignore the vector interpretation because as long as they are consistent then only matrix layout is significant.
If we accept 2, then there are only two possible combinations - either both host and shader are using `row-major` matrix layout or `column-major` layout.

This is simple, but is perhaps not the end of the story. First lets assume that we want our Slang code to be as portable as possible. As previously discussed for CUDA and C++/CPU targets Slang ignores the matrix layout settings - the matrix layout is *always* `row-major`.

Second lets consider performance. The matrix layout in a host maths library is not arbitrary from a performance point of view. A performant host maths library will want to use SIMD instructions. With both x86/x64 SSE and ARM NEON SIMD it makes a performance difference which layout is used, depending on if `column` or `row` is the *preferred* vector interpretation. If the `row` vector interpretation is preferred, it is most performant to have `row-major` matrix layout. Conversely if `column` vector interpretation is preferred `column-major` matrix will be the most performant.

The performance difference comes down to a SIMD implementation having to do a transpose if the layout doesn't match the preferred vector interpretation. 

If we put this all together - best performance, consistency between vector interpretation and platform independence we get:

1) Consistency : Same vector interpretation in shader and host code
2) Platform independence: Kernel uses `row-major` matrix layout
3) Performance: Host vector interpretation should match host matrix layout

The only combination that fulfills all aspects is `row-major` matrix layout and `row` vector interpretation for both host and kernel.

It's worth noting that for targets that honor the default matrix layout - that setting can act like a toggle transposing a matrix layout. If for some reason the combination of choices leads to inconsistent vector transforms, an implementation can perform this transform in *host* code at the boundary between host and the kernel. This is not the most performant or convenient scenario, but if supported in an implementation it could be used for targets that do not support kernel matrix layout settings. 

If only targeting platforms that honor matrix layout, there is more flexibility, our constraints are

1) Consistency : Same vector interpretation in shader and host code
2) Performance: Host vector interpretation should match host matrix layout

Then there are two combinations that work

1) `row-major` matrix layout for host and kernel, and `row` vector interpretation.
2) `column-major` matrix layout for host and kernel, and `column` vector interpretation.

If the host maths library is not performance orientated, it may be arbitrary from a performance point of view if a `row` or `column` vector interpretation is used. In that case assuming shader and host vector interpretation is the same it is only important that the kernel and maths library matrix layout match.

Another way of thinking about these combinations is to think of each change in `row-major`/`column-major` matrix layout and `row`/`column` vector interpretation is a transpose. If there are an *even* number of flips then all the transposes cancel out. Therefore the following combinations work

| Host Vector | Kernel Vector | Host Mat Layout | Kernel Mat Layout 
|-------------|---------------|-----------------|------------------
|   Row       |    Row        |    Row          |    Row
|   Row       |    Row        |    Column       |    Column
|   Column    |    Column     |    Row          |    Row
|   Column    |    Column     |    Column       |    Column
|   Row       |    Column     |    Row          |    Column
|   Row       |    Column     |    Column       |    Row
|   Column    |    Row        |    Row          |    Column
|   Column    |    Row        |    Column       |    Row

To be clear 'Kernel Mat Layout' is the shader matrix layout setting. As previously touched upon, if it is not possible to use the setting (say because it is not supported on a target), then doing a transpose at the host/kernel boundary can fix the issue. 

Matrix Layout
-------------

The above discussion is largely around 4x4 32-bit element matrices. For graphics APIs such as Vulkan, GL, and D3D there are typically additional restrictions for matrix layout. One restriction is for 16 byte alignment between rows (for `row-major` layout) and columns (for `column-major` layout). 

More CPU-like targets such as CUDA and C++/CPU do not have this restriction, and all elements are consecutive. 

This being the case only the following matrix types/matrix layouts will work across all targets. (Listed in the HLSL convention of RxC). 
 
* 1x4 `row-major` matrix layout
* 2x4 `row-major` matrix layout
* 3x4 `row-major` matrix layout
* 4x4 `row-major` matrix layout

These are all 'row-major' because as previously discussed currently only `row-major` matrix layout works across all targets currently.

NOTE! This only applies to matrices that are transferred between host and kernel - any matrix size will work appropriately for variables in shader/kernel code for example.

The hosts maths library also plays a part here. The library may hold all elements consecutively in memory. If that's the case it will match the CPU/CUDA kernels, but will only work on 'graphics'-like targets that match that layout for the size. 

For SIMD based host maths libraries it can be even more convoluted. If a SIMD library is being used that prefers `row` vector interpretation and therefore will have `row-major` layout it may for many sizes *not* match the CPU-like consecutive layout. For example a 4x3 - it will likely be packed with 16 byte row alignment. Additionally even if a matrix is packed in the same way it may not be the same size. For example a 3x2 matrix *may* hold the rows consecutively *but* be 16 bytes in size, as opposed to the 12 bytes that a CPU-like kernel will expect. 

If a SIMD based host maths library with graphics-like APIs are being used, there is a good chance (but certainly *not* guaranteed) that layout across non 4x4 sizes will match because SIMD typically implies 16 byte alignment. 

If your application uses matrix sizes that are not 4x4 across the host/kernel boundary and it wants to work across all targets, it is *likely* that *some* matrices will have to be converted at the boundary. This being the case, having to handle transposing matrices at the boundary is a less significant issue. 

In conclusion if your application has to perform matrix conversion work at the host/kernel boundary the previous observation about "best performance" implies `row-major` layout and `row` vector interpretation becomes somewhat mute.

Overriding default matrix layout
--------------------------------

Slang allows users to override default matrix layout with a compiler flag. This compiler flag can be specified during the creation of a `Session`:

```
slang::IGlobalSession* globalSession;
...
slang::SessionDesc slangSessionDesc = {};
slangSessionDesc.defaultMatrixLayoutMode = SLANG_MATRIX_LAYOUT_COLUMN_MAJOR;
...
slang::ISession* session;
globalSession->createSession(slangSessionDesc, &session);
```

This makes Slang treat all matrices as in `column-major` layout, and for example emitting `column_major` qualifier in resulting HLSL code.

Alternatively the default layout can be set by

* Including a `CompilerOptionName::MatrixLayoutColumn` or `CompilerOptionName::MatrixLayoutRow` entry in `SessionDesc::compilerOptionEntries`.
* Setting `-matrix-layout-row-major` or `-matrix-layout-column-major` command line options to `slangc`.


