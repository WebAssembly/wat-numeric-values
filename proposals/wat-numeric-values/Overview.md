# Wat Numeric Values

## Summary

* Currently, the data values in data segments in WAT can only be written in strings. 
For example:

    ```wat
    (data (offset (i32.const 0)) "1234")               ;; 3132 3334
    ```
    ```wat
    (data (offset (i32.const 0)) "\09\ab\cd\ef")       ;; 09ab cdef
    ```

* This proposal proposes another writing format that allows us to write integers and float values.

    ```wat
    (data (offset (i32.const 0))
        (f32 0.2 0.3 0.4)          ;; cdcc 4c3e 9a99 993e cdcc cc3e
    )
    ```
    ```wat
    (memory $1
        (data 
            (i8 1 2)               ;; 0102
            (i16 3 4)              ;; 0300 0400
        )
    )
    ```

* **Live Prototype:** https://wasmprop-numerical-data.netlify.app/wat2wasm/

* Compilers that have implemented this proposal:

    - [wasp](https://github.com/WebAssembly/wasp) (use `--enable-numeric-values` flag)
    - [Rust `wat`/`wast` crates](https://github.com/bytecodealliance/wasm-tools)

## Motivation

* Writing arbritary numeric values (integers and floats) to data segments is not simple.
We need to encode the data, add escape characters `\`, and write it as strings.

    For example, the following snippet is meant to write some float values to a data segment.

    ```wat
    (data (i32.const 0)
        "\00\00\c8\42"
        "\00\00\80\3f"
        "\00\00\00\3f"
    )
    ```

* If we ever need to review the numbers above, we cannot easily see the values without decoding it.

    `"\00\00\c8\42"` => `0x42c80000` => `100.0`

* x86-64 GCC and x86-64 Clang have [assembler directives](https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/html_chapter/as_7.html) that can be used to save arbritary numbers to data segments.

    Some of them are:

    ```s
    data:
    .ascii  "abcd"         #; 61 62 63 64
    .byte   1, 2, 3, 4     #; 01 02 03 04
    .2byte  5, 6           #; 05 00 06 00
    .4byte  0x89ABCDEF     #; EF CD AB 89
    .8byte  -1             #; FF FF FF FF FF FF FF FF
    .float  62.5           #; 00 00 7A 42
    .double 62.5           #; 00 00 00 00 00 40 4F 40
    ```

    In NASM, there are [pseudo-instructions](http://www.tortall.net/projects/yasm/manual/html/nasm-pseudop.html), for example:

    ```asm
    data:
    db          'abcd', 0x01, 2, 3, 4   ; 61 62 63 64 01 02 03 04
    dw          5, 6                    ; 05 00 06 00
    dd          62.5                    ; 00 00 7A 42
    dq          62.5                    ; 00 00 00 00 00 40 4F 40
    times 4 db  0xAB                    ; AB AB AB AB
    ```

    Those directives help programmers to write and see the data in the code directly in human readable format rather than the encoded format. This proposal proposes similar functionalities to WebAssembly Text Format.

## Overview

This proposal suggests a slight modification in the text format specification to accommodate writing numeric values in data segments.

### Text Format Spec Changes

The data value in data segments should accept both strings and list of numbers (numlist).

<pre>
data<sub>I</sub> ::= ‘(’ ‘data’ x:<a href="https://webassembly.github.io/spec/core/text/modules.html#text-memidx">memidx</a><sub>I</sub> ‘(’ ‘offset’ e:<a href="https://webassembly.github.io/spec/core/text/instructions.html#text-expr">expr</a><sub>I</sub> ‘)’ b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">dataval</a> ‘)’
                => { <a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">data</a> x', <a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">offset</a> e, <a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">init</a> b* }

dataval ::= (b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">datavalelem</a>)*       => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((b*)*)

datavalelem ::= b*:<a href="https://webassembly.github.io/spec/core/text/values.html#text-string">string</a>           => b*
             |  b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">numlist</a>           => b*
</pre>

Numlists denote sequences of bytes. They are enclosed in parentheses, start with a keyword to identify the type of the numbers, and followed by a list of numbers.

The numbers inside numlists represent their byte sequence using the respective encoding. They are encoded using two's complement encoding for integers and IEEE754 encoding for float values. Each numlist symbol represents the concatenation of the bytes of those numbers.

<pre>
numlist ::= ‘(’ ‘i8’  (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i8</a>)*  ‘)’      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i8</sub>(n))*)    (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i8</sub>(n))*) | < 2<sup>32</sup>)
        |  ‘(’ ‘i16’ (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i16</a>)* ‘)’      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i16</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i16</sub>(n))*)| < 2<sup>32</sup>)
        |  ‘(’ ‘i32’ (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i32</a>)* ‘)’      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i32</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i32</sub>(n))*)| < 2<sup>32</sup>)
        |  ‘(’ ‘i64’ (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i64</a>)* ‘)’      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i64</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i64</sub>(n))*)| < 2<sup>32</sup>)
        |  ‘(’ ‘f32’ (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-float">f32</a>)* ‘)’      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f32</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f32</sub>(n))*)| < 2<sup>32</sup>)
        |  ‘(’ ‘f64’ (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-float">f64</a>)* ‘)’      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f64</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f64</sub>(n))*)| < 2<sup>32</sup>)
</pre>

This new data value form should also be available in the inline data segment in the memory module.

<pre>
‘(’ ‘memory’ <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a><sup>?</sup> ‘(’ ‘data’ b<sup>n</sup>:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">dataval</a> ‘)’ ‘)’ ≡
    ‘(’ ‘memory’ <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>' m m ‘)’ ‘(’ ‘data’ <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>' ‘(’ ‘i32.const’ ‘0’ <a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">dataval</a> ‘)’
        (if <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>'=<a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a><sup>?</sup> ≠ 𝜖 ∨ <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>' <a href="https://webassembly.github.io/spec/core/text/values.html#text-id-fresh">fresh</a>, m=ceil(n/64Ki))
</pre>

#### SIMD v128

The SIMD proposal introduces [v128 value type](https://github.com/WebAssembly/simd/blob/master/proposals/simd/SIMD.md#simd-value-type).
When the SIMD feature is enabled, `datavalelem` should also accept a list of `v128` value type.

<pre>
datavalelem ::= b*:<a href="https://webassembly.github.io/spec/core/text/values.html#text-string">string</a>           => b*
             |  b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">numlist</a>           => b*
             |  b*:v128list         => b*

v128list ::= ‘(’ ‘v128’ (<i><a href="https://github.com/WebAssembly/simd/blob/master/proposals/simd/TextSIMD.md">v128 constant text format</a></i>)* ‘)’
</pre>

See the examples below for the usage.

### Usage Example

```wat
;; XYZ coordinate points 
(data (offset (i32.const 0))
    (f32 0.2 0.3 0.4)
    (f32 0.4 0.5 0.6)
    (f32 0.4 0.5 0.6)
)

;; Writing 1001st ~ 1010th prime number
(data (offset (i32.const 0x100))
    (i16 7927 7933 7937 7949 7951 7963 7993 8009 8011 8017)
)

;; PI
(data (offset (i32.const 0x200))
    (f64 3.14159265358979323846264338327950288)
)

;; v128 list (SIMD)
(data (offset (i32.const 0x300))
    (v128
        i32x4 0 0 0 0
        f64x2 1.0 1.5
    )
)

;; Inline in memory module
(memory (data (i8 1 2 3 4)))
```

### Execution Result Example

The conversion of numlists to data in data segments happens during the text format to binary format compilation. 

So, the following two snippents:

```wat
...
(memory 1)
(data (offset (i32.const 0))
    "abcd"
    (i16 -1)
    (f32 62.5)
)
...
```
```wat
...
(memory 1)
(data (offset (i32.const 0))
    "abcd"
    "\FF\FF"
    "\00\00\7a\42"
)
...
```

will output the same binary code:

```
...
; data segment header 0
0000010: 00                                        ; segment flags
0000011: 41                                        ; i32.const
0000012: 00                                        ; i32 literal
0000013: 0b                                        ; end
0000014: 0a                                        ; data segment size
; data segment data 0
0000015: 6162 6364 ffff 0000 7a42                  ; data segment data
000000e: 10                                        ; FIXUP section size
...
```

#### SIMD v128 Execution Example

The following three snippets will output the same binary code:
```wat
(data (offset (i32.const 0x00))
    (v128
        i32x4 0xA 0xB 0xC 0xD
        f64x2 1.0 0.5
    )
)
```
```wat
(data (offset (i32.const 0x00))
    (i32 0xA 0xB 0xC 0xD)
    (f64 1.0 0.5)
)
```
```wat
(data (offset (i32.const 0x00))
    "\0a\00\00\00\0b\00\00\00\0c\00\00\00\0d\00\00\00"
    "\00\00\00\00\00\00\f0?\00\00\00\00\00\00\e0?"
)
```

### Additional Information

#### Encoding

As previously described, the encoding of numbers inside numlists is two's complement for integers and IEEE754 for float, which is similar to the `t.store` memory instructions. This encoding is used to ensure that when we load the value from memory using the `load` memory instructions, the value will be consistent whether the data was stored by using `(data ... )` initialization or `t.store` instructions.

#### Data Alignment

Unaligned placements are allowed. For example:

```wat
...
(memory 1)
(data (offset (i32.const 0))
  (i8 1)    ;; will go to address 0
  (i16 2)   ;; will go to address 1
)
...
```
compiles to: `0102 00`

#### Out of Range Values

Out of range values should throw error during text format to binary format compilation.

```wat
(memory 1)
(data (offset (i32.const 0))
  (i8 256)        ;; Error
  (i8 -129)       ;; Error
)
```

#### Binary Format to Text Format Translation

The data segments in the compiled binary do not contain any information about their original form in WAT state.
Therefore, the translation from the binary format back to the text format will use the default string form.

#### Backward Compatibility

As the proposed grammar still accepts the string form, all existing WAT codes should work fine.

## Links

* Initial design issue discussion: https://github.com/WebAssembly/design/issues/1348

* First presentation in CG meeting: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-09.md

* Poll on advancing to phase 1: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-23.md
