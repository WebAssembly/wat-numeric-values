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

TODO

## Overview

TODO
