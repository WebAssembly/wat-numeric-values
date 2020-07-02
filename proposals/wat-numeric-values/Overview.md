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

    ([prototype repo](https://github.com/echamudi/wabt-wat-numeric-values/tree/patch-wat-numeric-values))

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

## Overview

This proposal suggests a slight modification in the text format specification to accommodate writing numeric values in data segments.

### Text Format Spec Changes

The data value in data segments should accept both strings and list of numbers (numvec).

<pre>
data<sub>I</sub> ::= ‘(’ ‘data’ x:<a href="https://webassembly.github.io/spec/core/text/modules.html#text-memidx">memidx</a><sub>I</sub> ‘(’ ‘offset’ e:<a href="https://webassembly.github.io/spec/core/text/instructions.html#text-expr">expr</a><sub>I</sub> ‘)’ b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">dataval</a> ‘)’
                => { <a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">data</a> x', <a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">offset</a> e, <a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">init</a> b* }

dataval ::= (b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">datavalelem</a>)*       => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((b*)*)

datavalelem ::= b*:<a href="https://webassembly.github.io/spec/core/text/values.html#text-string">string</a>           => b*
             |  b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">numvec</a>           => b*
</pre>

Numvecs denote sequences of bytes. They are enclosed in parentheses, starts with a keyword to identify the type of the numbers, and followed by the list of numbers.

The numbers inside numvecs are converted to the respective representation in bytes, and then concatenated into consecutive bytes.

<pre>
numvec ::= ‘(’ ‘i8’  (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i8</a>)*  ‘)’      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i8</sub>(n))*)    (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i8</sub>(n))*) | < 2<sup>32</sup>)
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
</pre>

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

;; Inline in memory module
(memory (data (i8 1 2 3 4)))
```

### Execution

The conversion of numvec to data in data segments happens during the text format to binary format compilation. 
Which means there is no change needed in the binary format spec, execution spec, or the structure spec.

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

#### Encoding

The encoding should use two's complement for integers and IEEE754 for float, which is similar to the `t.store` memory instructions.

This encoding is used to ensure that when we load the value from memory using the `load` memory instructions, the value will be consistent whether the data was stored by using `(data ... )` initialization or `t.store` instructions.

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

### Backward Compatibility

As the proposed grammar still accepts the string form, all existing WAT codes should work fine.

## Links

* Initial design issue discussion: https://github.com/WebAssembly/design/issues/1348

* First presentation in CG meeting: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-09.md

* Poll on advancing to phase 1: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-23.md
