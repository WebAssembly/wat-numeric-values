# Wat Numeric Values

## Summary

* Currently, the data segments can only be written in strings.

    ```wat
    (data (offset (i32.const 0)) "1234")               ;; 3132 3334
    ```
    ```wat
    (data (offset (i32.const 0)) "\09\ab\cd\ef")       ;; 09ab cdef
    ```

* This proposal proposes an alternative writing format in numeric values.

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

## Motivation

* Writing arbritary numeric values (integers and floats) to data segments is not easy.
We need to encode the data, add escape characters `\`, and write it as strings.

    For example, the following snippet tries to write some float values to a data segments.

    ```wat
    (data (i32.const 0)
        "\00\00\c8\42"
        "\00\00\80\3f"
        "\00\00\00\00"
    )
    ```

    And if we review the numbers, we cannot easily see the values without decoding it.
    `"\00\00\c8\42"` => `0x42c80000` => `100.0`

## Overview

TODO
