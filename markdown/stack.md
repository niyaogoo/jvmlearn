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