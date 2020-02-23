# 虚拟机内存访问

虚拟机内存访问提供了一些访问API, 以装饰模式实现不同的访问语义.
装饰模式是一组访问内存的属性, 提供以下一些属性:

* 内存排序方式
* 引用强度
* GC屏障
* 访问内存所属的位置
* 指针压缩与解压

实现方式使用了位操作、模板类、模板函数, 作为技术手段可以学习一下。

## 装饰位说明
所有的内存访问会带一个64位整型, 每一位代表一个访问属性, 属性根据目标分为不同的属性类, 
所有的属性类相交实现.

如内存排序共定义了6种属性, 每一种代表该次访问应该对应的内存排序类型.

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

## 访问接口实现

access翻译为访问, 一次访问可能是加载(load), 也可能是储存(store)
access.hpp提供了一些访问接口, 按照以下顺序执行

1. 首先提供一些默认的装饰属性, 将CV限定符去除(const volatile)

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

  //Oop访问, 该接口返回一个oop或narrowOop指针
  template <typename P>
  static inline AccessInternal::OopLoadProxy<P, decorators> oop_load(P* addr) {
    //验证oop方位装饰位, load_mo_decorators见上
    verify_oop_decorators<load_mo_decorators>();
    
    //提供一个代理类, 执行具体的访问, 这里是核心代码!!!, 见OopLoadProxy说明
    return AccessInternal::OopLoadProxy<P, decorators>(addr);
  }

  //验证实现
  template <DecoratorSet decorators>//模板类调用者指定装饰位
  template <DecoratorSet expected_decorators>//模板函数指定的预期装饰位
  void Access<decorators>::verify_decorators() {
    //检查是否有不允许的装饰位
    STATIC_ASSERT((~expected_decorators & decorators) == 0);
    const DecoratorSet barrier_strength_decorators = decorators & AS_DECORATOR_MASK;
    //屏障唯一
    STATIC_ASSERT(barrier_strength_decorators == 0 || (
      (barrier_strength_decorators ^ AS_NO_KEEPALIVE) == 0 ||
      (barrier_strength_decorators ^ AS_RAW) == 0 ||
      (barrier_strength_decorators ^ AS_NORMAL) == 0
    ));
    //引用属性唯一
    const DecoratorSet ref_strength_decorators = decorators & ON_DECORATOR_MASK;
    STATIC_ASSERT(ref_strength_decorators == 0 || (
      (ref_strength_decorators ^ ON_STRONG_OOP_REF) == 0 ||
      (ref_strength_decorators ^ ON_WEAK_OOP_REF) == 0 ||
      (ref_strength_decorators ^ ON_PHANTOM_OOP_REF) == 0 ||
      (ref_strength_decorators ^ ON_UNKNOWN_OOP_REF) == 0
    ));
    //内存排序属性唯一
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
}

```

accessBackend.hpp提供具体实现

``` c++
// hotspot/share/oops/accessBackend.hpp

//第一步, 设置默认的访问属性(decorators)
//OopLoadProxy作为Oop加载的代理类, P地址类型
template <typename P, DecoratorSet decorators>
class OopLoadProxy: public StackObj {
private:
  //访问地址
  P *const _addr;
public:
  OopLoadProxy(P* addr) : _addr(addr) {}

  //操作符重载oop, 赋值oop时调用, load见下
  inline operator oop() {
    return load<decorators | INTERNAL_VALUE_IS_OOP, P, oop>(_addr);
  }

  //操作符重载oop, 赋值narrowOop时调用, load见下
  inline operator narrowOop() {
    return load<decorators | INTERNAL_VALUE_IS_OOP, P, narrowOop>(_addr);
  }
};

//第一步, 去除cv限定符, P为地址类型, T为值类型
//注意参数P如果有volatile限定符会增加访问属性MO_VOLATILE
template <DecoratorSet decorators, typename P, typename T>
  inline T load(P* addr) {
    //验证decorators是否由INTERNAL_VALUE_IS_OOP标志,返回值必须是指针, 整形或浮点类型
    verify_types<decorators, T>();
    //去除参数的cv限定符
    typedef typename Decay<P>::type DecayedP;
    //验证如果有INTERNAL_VALUE_IS_OOP位则使用oop或narrowOop
    //如果没有则使用去除cv限定符的返回值类型
    typedef typename Conditional<HasDecorator<decorators, INTERNAL_VALUE_IS_OOP>::value,
                                 typename OopOrNarrowOop<T>::type,
                                 typename Decay<T>::type>::type DecayedT;
    //如果参数P有volatile限定符但是装饰位没有MO_VOLATILE位, 则使用默认的内存排序位.
    const DecoratorSet expanded_decorators = DecoratorFixup<
      (IsVolatile<P>::value && !HasDecorator<decorators, MO_DECORATOR_MASK>::value) ?
      (MO_VOLATILE | decorators) : decorators>::value;
    
    //调用load_reduce_types
    //expanded_decorators为上一步可能添加了volatile属性位的decorators
    //DecayedT为去除cv限定符的返回类型
    //将参数也去除cv限定符并const_cast
    //调用类型限定加载函数
    return load_reduce_types<expanded_decorators, DecayedT>(const_cast<DecayedP*>(addr));
  }

  //第二步, 限定类型(reduce type), P为地址类型， T为值类型, 
  //对于非oop(oop或narrowOop)， P与T类型必须完全一致, P的其他三种情况限定规则如下, 列为P, 行为T
  // |           | HeapWord  |   oop   | narrowOop |
  // |   oop     |  rt-comp  | hw-none |  hw-comp  |
  // | narrowOop |     x     |    x    |  hw-none  |
  // x表示不支持
  // rt-comp表示运行时检查oop是否需要压缩/解压
  // hw-non 表示运行前已知将不使用oop压缩/解压
  // hw-comp 表示运行前已知将使用oop压缩/解压
  //例如P为HeapWord， T只能是oop并且是运行时检查压缩
  //P为narrowOop，T可以是oop(hw-comp), 也可以是narrowOop(hw-none)
  //load_reduce_types, 指针压缩版本
  template <DecoratorSet decorators, typename T>
  inline typename OopOrNarrowOop<T>::type load_reduce_types(narrowOop* addr) {
    //增加属性位
    const DecoratorSet expanded_decorators = decorators | INTERNAL_CONVERT_COMPRESSED_OOP |
                                             INTERNAL_RT_USE_COMPRESSED_OOPS;
    //尝试运行前分发, OopOrNarrowOop指定oop或narrowOop, 这里将进入第三步                                         
    return PreRuntimeDispatch::load<expanded_decorators, typename OopOrNarrowOop<T>::type>(addr);
  }

  //..
  //..

  //注意这是PreRuntimeDispatch中的函数
  //第三步, 运行前分发, 这一步尝试避免运行时分发, 当属性位不含有INTERNAL_BT_BARRIER_ON_PRIMITIVES与INTERNAL_VALUE_IS_OOP时使用运行前分发
  //EnableIf元函数只有参数1为true时参数2才会被定义为type, 包括void
  template <DecoratorSet decorators, typename T>
  inline static typename EnableIf<
    !HasDecorator<decorators, AS_RAW>::value, T>::type
  load(void* addr) {
    //不含有INTERNAL_BT_BARRIER_ON_PRIMITIVES,INTERNAL_VALUE_IS_OOP属性位使用运行前分发
    if (is_hardwired_primitive<decorators>()) {
      //增加AS_RAW属性为, 调用另一个函数模板实例
      const DecoratorSet expanded_decorators = decorators | AS_RAW;
      return PreRuntimeDispatch::load<expanded_decorators, T>(addr);
    } else {
      //见第四步, RuntimeDispatch
      return RuntimeDispatch<decorators, T, BARRIER_LOAD>::load(addr);
    }
  }

  //硬件直接加载函数模板实例
  //CanHardwireRaws判断属性位是否是直接方位或指针压缩访问或运行时压缩访问
  template <DecoratorSet decorators, typename T>
  inline static typename EnableIf<
    HasDecorator<decorators, AS_RAW>::value && CanHardwireRaw<decorators>::value, T>::type
  load(void* addr) {
    typedef RawAccessBarrier<decorators & RAW_DECORATOR_MASK> Raw;
    if (HasDecorator<decorators, INTERNAL_VALUE_IS_OOP>::value) {
      return Raw::template oop_load<T>(addr);
    } else {
      return Raw::template load<T>(addr);
    }
  }

  //注意以下代码是RawAccessBarrier类
  template <typename T>
  static inline T load(void* addr) {
    return load_internal<decorators, T>(addr);
  }

  //内存排序属性位MO_ACQUIRE实现版本, acquire语义
  template <DecoratorSet decorators>
  template <DecoratorSet ds, typename T>
  inline typename EnableIf<
    HasDecorator<ds, MO_ACQUIRE>::value, T>::type
  RawAccessBarrier<decorators>::load_internal(void* addr) {
    //将跳转至atomic.hpp, atomic见另外笔记
    return Atomic::load_acquire(reinterpret_cast<const volatile T*>(addr));
  }

    //第四步, 运行时分发, 提供是需要要压缩/解压oop, 使用何种垃圾回收实现
  //实现模式为提供一个函数指针, 在第一次调用时会确认具体过程, 并再接下来的调用执行该过程
  //接第三步源码中的return RuntimeDispatch<decorators, T, BARRIER_LOAD>::load(addr);
  //对应第三步类模板实例, BARRIER_LOAD
  //为方便阅读将相关源码进行了合并

  template <DecoratorSet decorators, typename T>
  struct RuntimeDispatch<decorators, T, BARRIER_LOAD>: AllStatic {
    //访问函数指针, 注意这个函数指针指向load_init函数, 第一次调用调用后指针指向具体实现
    typedef typename AccessFunction<decorators, T, BARRIER_LOAD>::type func_t;
    static func_t _load_func = &load_init;
    
    static T load_init(void* addr) {
      //BarrierResolver见下面详解
      func_t function = BarrierResolver<decorators, func_t, BARRIER_LOAD>::resolve_barrier();
      //已找到具体实现， 替换_load_func指针, 接下来的所有调用都会执行上面指定的代码, 笔记中的oop_access_barrier
      _load_func = function;
      return function(addr);
    }

    //人畜无害
    static inline T load(void* addr) {
      return _load_func(addr);
    }
  };

  //对每个访问增加一些屏障, 比如压缩/解压屏障和其他在BarrierSet中定义屏障
  //oop版本
  template <DecoratorSet decorators, typename FunctionPointerT, BarrierType barrier_type>
  struct BarrierResolver: public AllStatic {
    template <DecoratorSet ds>
    //EnableIf第一个参数为true时第二个参数有效, false时编译失败因为没有对应type
    static typename EnableIf<
      HasDecorator<ds, INTERNAL_VALUE_IS_OOP>::value,
      FunctionPointerT>::type
    resolve_barrier_gc() {
      BarrierSet* bs = BarrierSet::barrier_set();
      assert(bs != NULL, "GC barriers invoked before BarrierSet is set");
      switch (bs->kind()) {
      //FOR_EACH_CONCRETE_BARRIER_SET_DO定义见下方
#define BARRIER_SET_RESOLVE_BARRIER_CLOSURE(bs_name)                    \
        case BarrierSet::bs_name: {                                     \
          return PostRuntimeDispatch<typename BarrierSet::GetType<BarrierSet::bs_name>::type:: \
            AccessBarrier<ds>, barrier_type, ds>::oop_access_barrier; \
        }                                                               \
        break;
        //BarrierSet::GetType<BarrierSet::bs_name>::type::AccessBarrier<ds>的g1实例版本见下方， 最后返回具体AccessBarrie中的oop_access_barrier函数指针
        FOR_EACH_CONCRETE_BARRIER_SET_DO(BARRIER_SET_RESOLVE_BARRIER_CLOSURE)
#undef BARRIER_SET_RESOLVE_BARRIER_CLOSURE

      default:
        fatal("BarrierSet AccessBarrier resolving not implemented");
        return NULL;
      };
    }

    //是否使用oop压缩
    static FunctionPointerT resolve_barrier_rt() {
      if (UseCompressedOops) {
        const DecoratorSet expanded_decorators = decorators | INTERNAL_RT_USE_COMPRESSED_OOPS;
        return resolve_barrier_gc<expanded_decorators>();
      } else {
        return resolve_barrier_gc<decorators>();
      }
    }

    //入口
    static FunctionPointerT resolve_barrier() {
      return resolve_barrier_rt();
    }
  }

  //第五步, 所有屏障已经确定, PostRuntime分发, 根据确定的地址类型再次模板是实例化, 比如oop或narrowOop
  //BARRIER_LOAD版本, 
  template <class GCBarrierType, DecoratorSet decorators>
  struct PostRuntimeDispatch<GCBarrierType, BARRIER_LOAD, decorators>: public AllStatic {
    template <typename T>
    static T access_barrier(void* addr) {
      return GCBarrierType::load_in_heap(reinterpret_cast<T*>(addr));
    }

    //oop函数, 对应RuntimeDispatcher中需要的返回值, 执行具体加载函数
    static oop oop_access_barrier(void* addr) {
      typedef typename HeapOopType<decorators>::type OopType;
      if (HasDecorator<decorators, IN_HEAP>::value) {
        return GCBarrierType::oop_load_in_heap(reinterpret_cast<OopType*>(addr));
      } else {
        return GCBarrierType::oop_load_not_in_heap(reinterpret_cast<OopType*>(addr));
      }
    }
  };

  //barrierSetConfig.hpp中定义的具体barrier set
  #define FOR_EACH_CONCRETE_BARRIER_SET_DO(f)          \
  f(CardTableBarrierSet)                             \
  EPSILONGC_ONLY(f(EpsilonBarrierSet))               \
  G1GC_ONLY(f(G1BarrierSet))                         \
  SHENANDOAHGC_ONLY(f(ShenandoahBarrierSet))         \
  ZGC_ONLY(f(ZBarrierSet))

```