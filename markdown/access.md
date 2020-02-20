# 虚拟机内存访问

虚拟机内存访问提供了一些访问API, 以装饰模式实现不同的访问语义.
装饰模式是一组访问内存的属性, 提供以下一些属性:

* 内存排序方式
* 引用强度
* GC屏障
* 访问内存所属的位置
* 指针压缩与解压

以上各个属性正交, 实现方式使用了位操作、模板类、模板函数, 作为技术手段可以学习一下。

``` c++
// hostpot/share/oops/accessDecorators.hpp
//装饰属性定义

//一个64位整形定义当前访问的语义
typedef uint64_t DecoratorSet;

//提供一个模板萃取帮助结构,在编译时确认是否包含指定的装饰
template <DecoratorSet decorators, DecoratorSet decorator>
struct HasDecorator: public IntegralConstant<bool, (decorators & decorator) != 0> {};

//默认空装饰, 不指定任何语义
const DecoratorSet DECORATORS_NONE                   = UCONST64(0);

// 内部装饰
// * INTERNAL_CONVERT_COMPRESSED_OOPS
// 需要开启指针压缩, 与UseCompressedOops全局参数必须被设置
// * INTERNAL_VALUE_IS_OOP: 普通对象
const DecoratorSet INTERNAL_CONVERT_COMPRESSED_OOP   = UCONST64(1) << 1;
const DecoratorSet INTERNAL_VALUE_IS_OOP             = UCONST64(1) << 2;

// GC屏障,暂不清楚如何工作
// * INTERNAL_BT_BARRIER_ON_PRIMITIVES: 原始空间gc屏障
// * INTERNAL_BT_TO_SPACE_INVARIANT: 常量空间不绑定gc屏障
const DecoratorSet INTERNAL_BT_BARRIER_ON_PRIMITIVES = UCONST64(1) << 3;
const DecoratorSet INTERNAL_BT_TO_SPACE_INVARIANT    = UCONST64(1) << 4;

// * INTERNAL_RT_USE_COMPRESSED_OOPS: 运行时指针压缩
const DecoratorSet INTERNAL_RT_USE_COMPRESSED_OOPS   = UCONST64(1) << 5;
//内部装饰掩码
const DecoratorSet INTERNAL_DECORATOR_MASK           = INTERNAL_CONVERT_COMPRESSED_OOP | INTERNAL_VALUE_IS_OOP |
                                                       INTERNAL_BT_BARRIER_ON_PRIMITIVES | INTERNAL_RT_USE_COMPRESSED_OOPS;

//内存排序装饰, 不同的装饰类型提供不同的排序强度, 更高一级的强度同时保证低强度的保证, 如MO_RELAXED同时有低一级的MO_VOLATILEB的保证
// === 储存 ===
//  * MO_UNORDERED (默认): 不保证.编译器或硬件可能重排指令
//  * MO_VOLATILE: c++ volatile, 储存不会被编译器重排, 但可能被硬件重排
//  * MO_RELAXED: relaxed语义, 保证线程内原子储存是顺序执行
//  * MO_RELEASE: release语义, 保证之前的内存访问可以被之后acquire语义的加载可观察,也就是loadstore storestore不能被重排
//  * MO_SEQ_CST: sc语义,sc存储前后都不重排
// === 加载 ===
//  * MO_UNORDERED (默认): 不保证
//  * MO_VOLATILE: c++ volatile, 储存不会被编译器重排, 但可能被硬件重排
//  * MO_RELAXED: relaxed语义, 原子读取
//  * MO_ACQUIRE: acquire语义, 保证之后的内存访问可以被之前release语义存储可观察,也就是loadload loadstore
//  * MO_SEQ_CST: sc语义
// === 原子比较替换 ===
//  * MO_RELAXED: relaxed语义比较替换
//  * MO_SEQ_CST: sc语义
// === 原子替换 ===
//  * MO_RELAXED: relaxed语义替换
//  * MO_SEQ_CST: sc语义
const DecoratorSet MO_UNORDERED      = UCONST64(1) << 6;
const DecoratorSet MO_VOLATILE       = UCONST64(1) << 7;
const DecoratorSet MO_RELAXED        = UCONST64(1) << 8;
const DecoratorSet MO_ACQUIRE        = UCONST64(1) << 9;
const DecoratorSet MO_RELEASE        = UCONST64(1) << 10;
const DecoratorSet MO_SEQ_CST        = UCONST64(1) << 11;
const DecoratorSet MO_DECORATOR_MASK = MO_UNORDERED | MO_VOLATILE | MO_RELAXED |
                                       MO_ACQUIRE | MO_RELEASE | MO_SEQ_CST;

// === 屏障强度装饰Barrier Strength Decorators ===
// * AS_RAW: 直接访问内存, 忽略其他语义, 不会设置gc屏障, 除了内存排序
//  - 访问oop指针将转化位直接内存访问不进行检查
//  - 访问narrowOop指针将转化为指针压缩/解压访问不进行检查
//  - 访问HeapWord指针在运行时使用以上两者之一
//  - 方位其他转化为直接内存访问不进行检查
// * AS_NO_KEEPALIVE: 屏障只能使用在oop引用, 不会将相关的对象保持存活,以一致性方式访问内存, 比如并发压缩, 保证能访问正确的地址
// * AS_NORMAL: 访问会以适当的gc屏障方式访问, 选择以何种屏障, 默认

//   注意直接访问内存只有在编译时开启屏障处理才会有效
const DecoratorSet AS_RAW                  = UCONST64(1) << 12;
const DecoratorSet AS_NO_KEEPALIVE         = UCONST64(1) << 13;
const DecoratorSet AS_NORMAL               = UCONST64(1) << 14;
const DecoratorSet AS_DECORATOR_MASK       = AS_RAW | AS_NO_KEEPALIVE | AS_NORMAL;

// === 引用强度修饰 ===
// 这些装饰只能使用在oop/narrowOop类型
// * ON_STRONG_OOP_REF: 强引用, 默认
// * ON_WEAK_OOP_REF: 弱引用
// * ON_PHANTOM_OOP_REF: 虚引用, 与jweak与weak oop一样
// * ON_UNKNOWN_OOP_REF: 未知引用, 不安全
const DecoratorSet ON_STRONG_OOP_REF  = UCONST64(1) << 15;
const DecoratorSet ON_WEAK_OOP_REF    = UCONST64(1) << 16;
const DecoratorSet ON_PHANTOM_OOP_REF = UCONST64(1) << 17;
const DecoratorSet ON_UNKNOWN_OOP_REF = UCONST64(1) << 18;
const DecoratorSet ON_DECORATOR_MASK  = ON_STRONG_OOP_REF | ON_WEAK_OOP_REF |
                                        ON_PHANTOM_OOP_REF | ON_UNKNOWN_OOP_REF;

// === 访问位置 ===
// 可以访问堆, 年轻代、老年代、不同的本地根，或本地非堆
// 位置对gc非常重要, 不同的位置gc采用不同的动作
// * IN_HEAP: 访问堆
// * IN_NATIVE: 访问本地非堆
const DecoratorSet IN_HEAP            = UCONST64(1) << 19;
const DecoratorSet IN_NATIVE          = UCONST64(1) << 20;
const DecoratorSet IN_DECORATOR_MASK  = IN_HEAP | IN_NATIVE;

// == 布尔标记装饰 ==
// * IS_ARRAY: 堆内分配的数组, gc会特殊处理
// * IS_DEST_UNINITIALIZED: 对SATB(snapshot at begin)非常重要, 未初始化地址
// * IS_NOT_NULL: 加快相关屏障速度, 如压缩oops
const DecoratorSet IS_ARRAY              = UCONST64(1) << 21;
const DecoratorSet IS_DEST_UNINITIALIZED = UCONST64(1) << 22;
const DecoratorSet IS_NOT_NULL           = UCONST64(1) << 23;

// == 数组拷贝装饰 ==
// * ARRAYCOPY_CHECKCAST: 不保证源地址的数组类型是目标地址的子类, 需要检查类型, 如果未设置, 则假设源与目标类型一样
// * ARRAYCOPY_DISJOINT: 两个数组不相交
// * ARRAYCOPY_ARRAYOF: 以arrayof的形式拷贝
// * ARRAYCOPY_ATOMIC: 以原子形式拷贝
// * ARRAYCOPY_ALIGNED: 地址以HeapWord对齐
const DecoratorSet ARRAYCOPY_CHECKCAST            = UCONST64(1) << 24;
const DecoratorSet ARRAYCOPY_DISJOINT             = UCONST64(1) << 25;
const DecoratorSet ARRAYCOPY_ARRAYOF              = UCONST64(1) << 26;
const DecoratorSet ARRAYCOPY_ATOMIC               = UCONST64(1) << 27;
const DecoratorSet ARRAYCOPY_ALIGNED              = UCONST64(1) << 28;
const DecoratorSet ARRAYCOPY_DECORATOR_MASK       = ARRAYCOPY_CHECKCAST | ARRAYCOPY_DISJOINT |
                                                    ARRAYCOPY_DISJOINT | ARRAYCOPY_ARRAYOF |
                                                    ARRAYCOPY_ATOMIC | ARRAYCOPY_ALIGNED;

// == 处理屏障装饰 ==
// * ACCESS_READ: 指定处理对象是只读访问, 将gc使用更有效的屏障
// * ACCESS_WRITE: 指定处理对象用于写访问
const DecoratorSet ACCESS_READ                    = UCONST64(1) << 29;
const DecoratorSet ACCESS_WRITE                   = UCONST64(1) << 30;

// 最后一位装饰位
const DecoratorSet DECORATOR_LAST = UCONST64(1) << 30;

namespace AccessInternal {
  
  //为元编程提供一些合并装饰
  template <DecoratorSet input_decorators>
  struct DecoratorFixup: AllStatic {
    //如果没有引用强度被指定, 使用强引用装饰位
    static const DecoratorSet ref_strength_default = input_decorators |
      (((ON_DECORATOR_MASK & input_decorators) == 0 && (INTERNAL_VALUE_IS_OOP & input_decorators) != 0) ?
       ON_STRONG_OOP_REF : DECORATORS_NONE);
    //如果没有内存排序被指定, 使用不指定内存排序装饰位
    static const DecoratorSet memory_ordering_default = ref_strength_default |
      ((MO_DECORATOR_MASK & ref_strength_default) == 0 ? MO_UNORDERED : DECORATORS_NONE);
    // If no barrier strength has been picked, normal will be used
    //如果没有屏障强度指定, 使用普通屏障装饰位
    static const DecoratorSet barrier_strength_default = memory_ordering_default |
      ((AS_DECORATOR_MASK & memory_ordering_default) == 0 ? AS_NORMAL : DECORATORS_NONE);
    static const DecoratorSet value = barrier_strength_default | BT_BUILDTIME_DECORATORS;
  };

  //与上面一样, 函数版实现
  inline DecoratorSet decorator_fixup(DecoratorSet input_decorators) {
    DecoratorSet ref_strength_default = input_decorators |
      (((ON_DECORATOR_MASK & input_decorators) == 0 && (INTERNAL_VALUE_IS_OOP & input_decorators) != 0) ?
       ON_STRONG_OOP_REF : DECORATORS_NONE);
    DecoratorSet memory_ordering_default = ref_strength_default |
      ((MO_DECORATOR_MASK & ref_strength_default) == 0 ? MO_UNORDERED : DECORATORS_NONE);
    DecoratorSet barrier_strength_default = memory_ordering_default |
      ((AS_DECORATOR_MASK & memory_ordering_default) == 0 ? AS_NORMAL : DECORATORS_NONE);
    DecoratorSet value = barrier_strength_default | BT_BUILDTIME_DECORATORS;
    return value;
  }
}
```

``` c++
// hotspot/share/oops/access.hpp
//该文件提供所有暴露的API

//Access提供所有的API, 使用0装饰
//注意类模板的装饰参数为调用者定义, 与verify*函数的expected_decorators的函数模板参数用途不一样.
//verify*函数的模板参数用于验证调用者提供的各装饰位是否正确,并且相关不能正交的位不能正交
template <DecoratorSet decorators = DECORATORS_NONE>
class Access: public AllStatic {
  
  //主要验证Access类模板参数的装饰不能正交的不正交, 比如一个访问操作不是同时作用于堆与根上
  //主验证函数, 其他verify*相关添加相关预期装饰位并调用该函数
  template <DecoratorSet expected_decorators>
  static void verify_decorators();

  //用于验证oop相关装饰位, 只允许AS,IN,非ON_UNKNOWN_OOP_REF的其他ON, 
  template <DecoratorSet expected_mo_decorators>
  static void verify_oop_decorators() {
    //只允许以下装饰位
    const DecoratorSet oop_decorators = AS_DECORATOR_MASK | IN_DECORATOR_MASK |
                                        (ON_DECORATOR_MASK ^ ON_UNKNOWN_OOP_REF) |//不允许ON_KNOWN_OOP_REF
                                        IS_ARRAY | IS_NOT_NULL | IS_DEST_UNINITIALIZED;
    verify_decorators<expected_mo_decorators | oop_decorators>();
  }
  //其他验证类似

  //访问加载型的装饰位, 允许以下装饰位
  static const DecoratorSet load_mo_decorators = MO_UNORDERED | MO_VOLATILE | MO_RELAXED | MO_ACQUIRE | MO_SEQ_CST;

  //== API ==
  //Oop访问
  template <typename P>
  static inline AccessInternal::OopLoadProxy<P, decorators> oop_load(P* addr) {
    //验证oop方位装饰位, load_mo_decorators见上
    verify_oop_decorators<load_mo_decorators>();
    
    return AccessInternal::OopLoadProxy<P, decorators>(addr);
  }
}

//验证实现
template <DecoratorSet decorators>//模板类调用者指定装饰位
template <DecoratorSet expected_decorators>//模板函数指定的预期装饰位
void Access<decorators>::verify_decorators() {
  //检查是否有不允许的装饰位
  STATIC_ASSERT((~expected_decorators & decorators) == 0);
  const DecoratorSet barrier_strength_decorators = decorators & AS_DECORATOR_MASK;
  //所有屏障位不能正交
  STATIC_ASSERT(barrier_strength_decorators == 0 || (
    (barrier_strength_decorators ^ AS_NO_KEEPALIVE) == 0 ||
    (barrier_strength_decorators ^ AS_RAW) == 0 ||
    (barrier_strength_decorators ^ AS_NORMAL) == 0
  ));
  //所有引用位不能正交
  const DecoratorSet ref_strength_decorators = decorators & ON_DECORATOR_MASK;
  STATIC_ASSERT(ref_strength_decorators == 0 || (
    (ref_strength_decorators ^ ON_STRONG_OOP_REF) == 0 ||
    (ref_strength_decorators ^ ON_WEAK_OOP_REF) == 0 ||
    (ref_strength_decorators ^ ON_PHANTOM_OOP_REF) == 0 ||
    (ref_strength_decorators ^ ON_UNKNOWN_OOP_REF) == 0
  ));
  //所有内存排序不能正交
  const DecoratorSet memory_ordering_decorators = decorators & MO_DECORATOR_MASK;
  STATIC_ASSERT(memory_ordering_decorators == 0 || (
    (memory_ordering_decorators ^ MO_UNORDERED) == 0 ||
    (memory_ordering_decorators ^ MO_VOLATILE) == 0 ||
    (memory_ordering_decorators ^ MO_RELAXED) == 0 ||
    (memory_ordering_decorators ^ MO_ACQUIRE) == 0 ||
    (memory_ordering_decorators ^ MO_RELEASE) == 0 ||
    (memory_ordering_decorators ^ MO_SEQ_CST) == 0
  ));
  //无法同时在堆与本地内存
  const DecoratorSet location_decorators = decorators & IN_DECORATOR_MASK;
  STATIC_ASSERT(location_decorators == 0 || (
    (location_decorators ^ IN_NATIVE) == 0 ||
    (location_decorators ^ IN_HEAP) == 0
  ));
}

```