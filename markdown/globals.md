# 全局变量定义

## globals
全局变量主要定义在globals.hpp文件中, 宏名称后缀_FLAGS, 分级如下:

* ALL_FLAGS: 包含其他所有FLAG
  * VM_FLAGS
    * RUNTIME_FLAGS: 虚拟机运行时变量
    * GC_FLAGS: 垃圾回收变量
      * GC_CMS_FLAGS:
      * GC_EPSILON_FLAGS:
      * GC_G1_FLAGS: 
      * GC_PARALLEL_FLAGS:
      * GC_SERIAL_FLAGS:
      * GC_SHENANDOAH_FLAGS:
      * GC_Z_FLAGS:
  * RUNTIME_OS_FLAGS: 操作系统相关变量
  * JVMCI_FLAGS: 开启JVMCI编译器后有效
  * C1_FLAGS: 开启C1编译器有效
  * C2_FLAGS: 开启C2编译器有效
  * ARCH_FLAGS: 硬件相关

所有变量由ALL_FLAGS宏扩展后生成
```c++
// globals_shared.hpp

#define ALL_FLAGS(            \
    develop,                  \
    develop_pd,               \
    product,                  \
    product_pd,               \
    diagnostic,               \
    diagnostic_pd,            \
    experimental,             \
    notproduct,               \
    manageable,               \
    product_rw,               \
    lp64_product,             \
    range,                    \
    constraint,               \
    writeable)
```
参数develop,develop_pd等定义了变量是否可以设置与可见, 比如develop在开发环境中可以设置也可见, 但是在生产环境是常量.参考如下宏
```c++

// Interface macros
#define DECLARE_PRODUCT_FLAG(type, name, value, doc)      extern "C" type name;
#define DECLARE_PD_PRODUCT_FLAG(type, name, doc)          extern "C" type name;
#define DECLARE_DIAGNOSTIC_FLAG(type, name, value, doc)   extern "C" type name;
#define DECLARE_PD_DIAGNOSTIC_FLAG(type, name, doc)       extern "C" type name;
#define DECLARE_EXPERIMENTAL_FLAG(type, name, value, doc) extern "C" type name;
#define DECLARE_MANAGEABLE_FLAG(type, name, value, doc)   extern "C" type name;
#define DECLARE_PRODUCT_RW_FLAG(type, name, value, doc)   extern "C" type name;
#ifdef PRODUCT
#define DECLARE_DEVELOPER_FLAG(type, name, value, doc)    const type name = value;
#define DECLARE_PD_DEVELOPER_FLAG(type, name, doc)        const type name = pd_##name;
#define DECLARE_NOTPRODUCT_FLAG(type, name, value, doc)   const type name = value;
#else
#define DECLARE_DEVELOPER_FLAG(type, name, value, doc)    extern "C" type name;
#define DECLARE_PD_DEVELOPER_FLAG(type, name, doc)        extern "C" type name;
#define DECLARE_NOTPRODUCT_FLAG(type, name, value, doc)   extern "C" type name;
#endif // PRODUCT
```
在生产环境下以const修饰

## jvmFlag
在源代码中可以看到有FLAG_IS_DEFAULT这样的调用, 用来判断当前变量的状态是否是默认值还是被优化过等, 其实内部调用的是JVMFLagEx::is_default()
```c++
// hotspot/share/runtime/flags/jvmFlag.cpp

JVMFlag* JVMFlag::flags = flagTable;
//..
//..
bool JVMFlagEx::is_default(JVMFlagsEnum flag) {
  return flag_from_enum(flag)->is_default();
}

// Returns the address of the index'th element
JVMFlag* JVMFlagEx::flag_from_enum(JVMFlagsEnum flag) {
  assert((size_t)flag < JVMFlag::numFlags, "bad command line flag index");
  return &JVMFlag::flags[flag];
}
```
注意其中的JVMFlag::flagTable生成方式
```c++
#define RUNTIME_PRODUCT_FLAG_STRUCT(     type, name, value, doc) { #type, XSTR(name), &name,         NOT_PRODUCT_ARG(doc) JVMFlag::Flags(JVMFlag::DEFAULT | JVMFlag::KIND_PRODUCT) },
#define RUNTIME_PD_PRODUCT_FLAG_STRUCT(  type, name,        doc) { #type, XSTR(name), &name,         NOT_PRODUCT_ARG(doc) JVMFlag::Flags(JVMFlag::DEFAULT | JVMFlag::KIND_PRODUCT | JVMFlag::KIND_PLATFORM_DEPENDENT) },
#define RUNTIME_DIAGNOSTIC_FLAG_STRUCT(  type, name, value, doc) { #type, XSTR(name), &name,         NOT_PRODUCT_ARG(doc) JVMFlag::Flags(JVMFlag::DEFAULT | JVMFlag::KIND_DIAGNOSTIC) },
// ..
// ..

static JVMFlag flagTable[] = {
  VM_FLAGS(RUNTIME_DEVELOP_FLAG_STRUCT, \
           RUNTIME_PD_DEVELOP_FLAG_STRUCT, \
           RUNTIME_PRODUCT_FLAG_STRUCT, \
           RUNTIME_PD_PRODUCT_FLAG_STRUCT, \
           RUNTIME_DIAGNOSTIC_FLAG_STRUCT, \
           RUNTIME_PD_DIAGNOSTIC_FLAG_STRUCT, \
           RUNTIME_EXPERIMENTAL_FLAG_STRUCT, \
           RUNTIME_NOTPRODUCT_FLAG_STRUCT, \
           RUNTIME_MANAGEABLE_FLAG_STRUCT, \
           RUNTIME_PRODUCT_RW_FLAG_STRUCT, \
           RUNTIME_LP64_PRODUCT_FLAG_STRUCT, \
           IGNORE_RANGE, \
           IGNORE_CONSTRAINT, \
           IGNORE_WRITEABLE)
   RUNTIME_OS_FLAGS(RUNTIME_DEVELOP_FLAG_STRUCT, \
// ..
// ..
```
其实类似于ALL_FLAG宏, 扩展后生成一个JVMFlag的结构, 内部的_addr指向ALL_FLAG生成的变量, 并标记Flags为DEFAULT
```c++
// hotspot/share/runtime/flags/jvmFlag.cpp
struct JVMFlag {
  //..
  //..

  const char* _type;
  const char* _name;
  void* _addr;
  NOT_PRODUCT(const char* _doc;)
  Flags _flags;
```