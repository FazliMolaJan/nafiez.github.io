---
layout: post
title:  "catdoc & xls2csv Buffer Overflow"
date:   2018-09-19 03:39:03 +0700
categories: [security, overflow]
---

Details
-------
There's an buffer & heap overflow found during fuzzing **catdoc** and **xls2csv** program. Reported to Ubuntu security team as the issue are valid with latest repo 
catdoc_0.94.3~git20160113.dbc9ec6+dfsg-1_i386. No fixed until this date.

catdoc ASAN output:
```
==32764==ERROR: AddressSanitizer: global-buffer-overflow on address 0x08071990 at pc 0x08050bc2 bp 0xbf86a2c8 sp 0xbf86a2b8
READ of size 1 at 0x08071990 thread T0
    #0 0x8050bc1 in format_rk /home/john/catdoc/src/xlsparse.c:716
    #1 0x805290d in process_item /home/john/catdoc/src/xlsparse.c:325
    #2 0x8054206 in do_table /home/john/catdoc/src/xlsparse.c:116
    #3 0x804a7fb in main /home/john/catdoc/src/xls2csv.c:167
    #4 0xb776e636 in __libc_start_main (/lib/i386-linux-gnu/libc.so.6+0x18636)
    #5 0x804b5db  (/usr/local/bin/xls2csv+0x804b5db)

0x08071990 is located 48 bytes to the left of global variable 'buffer' defined in 'charsets.c:291:14' (0x80719c0) of size 7
0x08071990 is located 0 bytes to the right of global variable 'rec' defined in 'xlsparse.c:22:22' (0x806d340) of size 18000
SUMMARY: AddressSanitizer: global-buffer-overflow /home/john/catdoc/src/xlsparse.c:716 format_rk
Shadow bytes around the buggy address:
  0x2100e2e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x2100e2f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x2100e300: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x2100e310: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x2100e320: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x2100e330: 00 00[f9]f9 f9 f9 f9 f9 07 f9 f9 f9 f9 f9 f9 f9
  0x2100e340: 04 f9 f9 f9 f9 f9 f9 f9 00 f9 f9 f9 f9 f9 f9 f9
  0x2100e350: 04 f9 f9 f9 f9 f9 f9 f9 04 f9 f9 f9 f9 f9 f9 f9
  0x2100e360: 04 f9 f9 f9 f9 f9 f9 f9 04 f9 f9 f9 f9 f9 f9 f9
  0x2100e370: 04 f9 f9 f9 f9 f9 f9 f9 04 f9 f9 f9 f9 f9 f9 f9
  0x2100e380: 04 f9 f9 f9 f9 f9 f9 f9 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
==32764==ABORTING
```

xls2csv ASAN output:
```
==4172==ERROR: AddressSanitizer: heap-buffer-overflow on address 0xb5f01618 at pc 0x0805f499 bp 0xbfbb1c68 sp 0xbfbb1c58
READ of size 1 at 0xb5f01618 thread T0
    0 0x805f498 in getlong /home/john/catdoc/src/numutils.c:22
    1 0x8064aae in ole_init /home/john/catdoc/src/ole.c:254
    2 0x8050f8b in analyze_format /home/john/catdoc/src/analyze.c:58
    3 0x804aab4 in main /home/john/catdoc/src/catdoc.c:180
    4 0xb7891636 in __libc_start_main (/lib/i386-linux-gnu/libc.so.6+0x18636)
    5 0x804ba7b  (/usr/local/bin/catdoc+0x804ba7b)

0xb5f01618 is located 6 bytes to the right of 2-byte region [0xb5f01610,0xb5f01612)
allocated by thread T0 here:
    0 0xb7ac5dee in malloc (/usr/lib/i386-linux-gnu/libasan.so.2+0x96dee)
    1 0xb78ee2c5 in __strdup (/lib/i386-linux-gnu/libc.so.6+0x752c5)

SUMMARY: AddressSanitizer: heap-buffer-overflow /home/john/catdoc/src/numutils.c:22 getlong
Shadow bytes around the buggy address:
  0x36be0270: fa fa 02 fa fa fa 05 fa fa fa 03 fa fa fa 02 fa
  0x36be0280: fa fa 03 fa fa fa 03 fa fa fa 03 fa fa fa 03 fa
  0x36be0290: fa fa 02 fa fa fa 02 fa fa fa 02 fa fa fa 02 fa
  0x36be02a0: fa fa 04 fa fa fa 03 fa fa fa 03 fa fa fa 03 fa
  0x36be02b0: fa fa 03 fa fa fa 02 fa fa fa 02 fa fa fa 02 fa
=>0x36be02c0: fa fa 02[fa]fa fa 02 fa fa fa 02 fa fa fa 02 fa
  0x36be02d0: fa fa 02 fa fa fa 02 fa fa fa 02 fa fa fa 02 fa
  0x36be02e0: fa fa 02 fa fa fa 02 fa fa fa 02 fa fa fa 03 fa
  0x36be02f0: fa fa 02 fa fa fa 02 fa fa fa 02 fa fa fa 02 fa
  0x36be0300: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x36be0310: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
==4172==ABORTING
```

Similar bug from https://catdocbugs.neocities.org/ when fuzzing xls2csv
```
- bug17-numutils-asan.doc
- bug20-numutils-asan.doc
- bug26-numutils-asan.doc
```
