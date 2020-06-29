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

* **Prototype:** https://wasmprop-numerical-data.netlify.app/wat2wasm/

## Motivation

* Writing arbritary numeric values (integers and floats) to data segments is not easy.
We need to encode the data, add escape characters `\`, and write it as strings.

    For example, the following snippet is meant to write some float values to a data segment.

    ```wat
    (data (i32.const 0)
        "\00\00\c8\42"
        "\00\00\80\3f"
        "\00\00\00\3f"
    )
    ```

* If we want to review the numbers above, we cannot easily see the values without decoding it.

    `"\00\00\c8\42"` => `0x42c80000` => `100.0`

    It wouldn't be a problem if we are allowed to write the float values directly, for example:

    ```wat
    (data (i32.const 0)
        (f32 100.0 1.0 0.5)
    )
    ```

## Overview

### Grammar Update

The data value in data segments should accept both strings and list of numbers (numvec).

```ebnf
data ::= '(' 'data' memidx '(' 'offset' expr ')' dataval* ')'

dataval ::= string
        | numvec
```

Numvec is a list of integers or float values. It is defined as follows:

```ebnf
numvec ::= '(' 'i8'  i8* ')'
        | '(' 'i16' i16* ')'
        | '(' 'i32' i32* ')'
        | '(' 'i64' i64* ')'

        | '(' 'f32' f32* ')'
        | '(' 'f64' f64* ')'
```

This data value form should also be available in the inline data segment in the memory module.

```ebnf
mem ::= '(' 'memory' id '(' 'data' dataval* ')' ')'
      |  ...
```

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

The conversion of numvec to data in data segments happens during the wat2wasm compilation. 
Which means there is no change needed in the binary format spec or the structure spec.

So, the following two snippents:

```wat
...
(memory 1)
(data (offset (i32.const 0))
  "abc"
  (i16 -1)
  (f32 62.5)
)
...
```
```wat
...
(memory 1)
(data (offset (i32.const 0))
  "abc"
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
0000012: 12                                        ; i32 literal
0000013: 0b                                        ; end
0000014: 0a                                        ; data segment size
; data segment data 0
0000015: 6162 6364 ffff 0000 7a42                  ; data segment data
000000e: 10                                        ; FIXUP section size
...
```

#### Encoding

The encoding should use two's complement for integers and IEEE754 for float, which is similar to the `t.store` memory instructions.

This encoding is used to make sure that when we load the value from memory using the `load` memory instructions, the value will be consistant whether the data was stored by using `(data ... )` initialization or `t.store` instructions.

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

Out of range values should throw error during wat2wasm compilation.

```wat
(memory 1)
(data (offset (i32.const 0))
  (i8 256)        ;; Error
  (i8 -129)       ;; Error
)
```

#### wasm2wat translation

The data segments in the compiled binary do not contain any information about their original form in WAT state.
Therefore, the translation from the binary format back to the text format will use the default string form.

### Backward Compatibility

As the proposed grammar still accepts the string form, all existing WAT codes should work fine.

## Links

* Initial design issue discussion: https://github.com/WebAssembly/design/issues/1348

* First presentation in CG meeting: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-09.md

* Poll on advancing to phase 1: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-23.md
