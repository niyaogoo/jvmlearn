# 性能数据

> hotspot/share/runtime/perfData.hpp
* PerfData提供创建、访问、更新性能数据, 一般存放在共享内存中.
* PerfData提供继承性, 初始化两种
  * PerfLong: 以long保存
  * PerfByteArray: 以c++字符数组保存



## perfData共有三种
* Constants: 一次性写
* Variables: 可多次修改
* Counters: 单调性增加或减少