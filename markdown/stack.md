# 虚拟机栈结构

```c++
  // Stack overflow support
  //
  //  (small addresses)低地址
  //
  //  --  <-- stack_end()                   ---栈顶
  //  |                                      |
  //  |  red pages                           |红区
  //  |                                      |
  //  --  <-- stack_red_zone_base()          |红区基地址
  //  |                                      |
  //  |                                     guard守护区
  //  |  yellow pages                       zone黄区
  //  |                                      |
  //  |                                      |
  //  --  <-- stack_yellow_zone_base()       |黄区基地址
  //  |                                      |
  //  |                                      |
  //  |  reserved pages                      |保留区
  //  |                                      |
  //  --  <-- stack_reserved_zone_base()    ---      ---保留区基地址
  //                                                 /|\  shadow     <--  stack_overflow_limit() (somewhere in here)影子区, stack_overflow_limit在这块
  //                                                  |   zone
  //                                                 \|/  size
  //  some untouched memory                          ---留白区
  //
  //影子区
  //  --
  //  |
  //  |  shadow zone
  //  |
  //  --
  //  x    frame n 栈帧n
  //  --
  //  x    frame n-1 栈帧n-1
  //  x
  //  --
  //  ...
  //
  //  --
  //  x    frame 0 栈帧0
  //  --  <-- stack_base() 栈底
  //
  //  (large addresses)高地址
  //
```

## Call stub
从C调用Java入口在hotspot/share/runtime/javaCalls.cpp#JavaCalls::call_helper

``` c++
// Call stubs are used to call Java from C
  //
  // Linux Arguments:
  //    c_rarg0:   call wrapper address                   address
  //    c_rarg1:   result                                 address
  //    c_rarg2:   result type                            BasicType
  //    c_rarg3:   method                                 Method*
  //    c_rarg4:   (interpreter) entry point              address
  //    c_rarg5:   parameters                             intptr_t*
  //    16(rbp): parameter size (in words)              int
  //    24(rbp): thread                                 Thread*
  //
  //     [ return_from_Java     ] <--- rsp
  //     [ argument word n      ]
  //      ...
  // -12 [ argument word 1      ]
  // -11 [ saved r15            ] <--- rsp_after_call
  // -10 [ saved r14            ]
  //  -9 [ saved r13            ]
  //  -8 [ saved r12            ]
  //  -7 [ saved rbx            ]
  //  -6 [ call wrapper         ]
  //  -5 [ result               ]
  //  -4 [ result type          ]
  //  -3 [ method               ]
  //  -2 [ entry point          ]
  //  -1 [ parameters           ]
  //   0 [ saved rbp            ] <--- rbp
  //   1 [ return address       ]
  //   2 [ parameter size       ]
  //   3 [ thread               ]
```