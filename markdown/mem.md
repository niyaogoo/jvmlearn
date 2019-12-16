# 虚拟机内存分配

## ResourceObj
ResourceObj用于存放对象, 可以存放在三种内存类型上
1. STACK_OR_EMBEDDED: 栈
2. RESOURCE_AREA: 以Arena的形式存放
3. C_HEAP: 直接存放在C堆

变量_allocation_t:用来校验ResourceObj对象地址与类型
* _allocation_t[0]:校验和, 地址与类型反码求和
* _allocation_t[1]:存放类型

## 虚拟机内存类型
### ResourceArea

### 物理机配置
> 操作系统:linux x86-64, 内存16g

### 相关参数
* hotspot/share/utilities/globalDefinitions.hpp
  * UnscaledOopHeapMax: 不使用指针压缩的内存边界, 4GB;
  * OopEncodingHeapMax: 使用指针压缩的内存边界, 32G;
  * HeapBaseMinAddress: 堆最小地址, 虚拟机会在2G-32G中尝试分配堆内存;

如果堆起始地址加上堆所需内存小于4G, 则不启用指针压缩

### MaxHeapSize
1. MaxHeapSize在64bit的机器上默认为96M * 13 / 10并以8向下对齐.
2. 如未设置-Xmx,虚拟机会在Arguments::set_heap_size中优化当前堆内存参数, 如果物理内存的50%未达到MaxHeapSize的默认值, 取物理内存的50%作为MaxHeapSize, 如果达到则使用物理内存的25%作为MaxHeapSize

### 分配虚拟堆
1. 未通过任何方式设置过HeapBaseMinAddress时将在内存位置的0x80000000处分配虚拟堆
1. 

