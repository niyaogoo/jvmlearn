# 内存对象说明

## Chunk
> hotspot/share/memory/arena.hpp#Chunk

链表形式管理原生内存, 长度有几种类型

* slack: 在64位系统下40, 否则20, 默认长度, 比2**k稍微小一些, 防止使用伙伴算法系统的内存分配实现, [TODO 不是太理解]
* tiny_size: 256 - slack, 第一个chunk的长度
* init_size: 1K - slack, 第一个chunk的长度, 也叫small size
* medium_size: 10K - slack, 中型长度
* size: 32K - slack, 第一个chunk后的长度
* non_pool_size: init_size + 32, 非以上长度

除了non_pool_size之外其他4个长度类型都有对应的ChunkPool

### new与delete
``` c++

void* Chunk::operator new (size_t requested_size, AllocFailType alloc_failmode, size_t length) throw() {
  // requested_size应该是sizeof(Chunk), 考虑到Arena使用的是ARENA_ALIGN宏对齐(sizeof(Chunk)后16字节对齐(64bit系统))
  assert(ARENA_ALIGN(requested_size) == aligned_overhead_size(), "Bad alignment");
  //length确定需要分配到哪一个ChunkPool中
  size_t bytes = ARENA_ALIGN(requested_size) + length;
  switch (length) {
   case Chunk::size:        return ChunkPool::large_pool()->allocate(bytes, alloc_failmode);
   case Chunk::medium_size: return ChunkPool::medium_pool()->allocate(bytes, alloc_failmode);
   case Chunk::init_size:   return ChunkPool::small_pool()->allocate(bytes, alloc_failmode);
   case Chunk::tiny_size:   return ChunkPool::tiny_pool()->allocate(bytes, alloc_failmode);
   default: {
     //直接malloc
     void* p = os::malloc(bytes, mtChunk, CALLER_PC);
     if (p == NULL && alloc_failmode == AllocFailStrategy::EXIT_OOM) {
       vm_exit_out_of_memory(bytes, OOM_MALLOC_ERROR, "Chunk::new");
     }
     return p;
   }
  }
}

void Chunk::operator delete(void* p) {
  Chunk* c = (Chunk*)p;
  //根据不同的len在不同的pool中删除
  switch (c->length()) {
   case Chunk::size:        ChunkPool::large_pool()->free(c); break;
   case Chunk::medium_size: ChunkPool::medium_pool()->free(c); break;
   case Chunk::init_size:   ChunkPool::small_pool()->free(c); break;
   case Chunk::tiny_size:   ChunkPool::tiny_pool()->free(c); break;
   default:
    //linux中使用pthread_mutex锁, 支持可重入
     ThreadCritical tc;
     os::free(c);
  }
}
```

### 删除方法
``` c++

//删除当前与之后的所有chunk
void Chunk::chop() {
  Chunk *k = this;
  while( k ) {
    Chunk *tmp = k->next();
    //如果标记了ZapResourceArea则设置该块内存为0xAB, ZapResourceArea在globals.hpp中设置, debug时打开
    if (ZapResourceArea) memset(k->bottom(), badResourceValue, k->length());
    delete k;//删除, 注意在Chunk的new与delete操作符已重载
    k = tmp;//指向next
  }
}

```

## ChunkPool
> hotspot/share/memory/arena.cpp#ChunkPool

内存类型安全的Chunk池, 减少malloc/free的抖动, 未使用Mutex锁, 因为pool在线程初始化之前就已使用

以链表实现原始内存块

## Area
> hotspot/share/memory/area.hpp#Area

负责内存的快速分配, 管理Chunk
### 配置
``` c++
  MEMFLAGS    _flags;           //内存追踪标记

  Chunk *_first;                //第一个chunk
  Chunk *_chunk;                //当前chunk
  char *_hwm, *_max;            //高水位标记与当前chunk最大值
```

### 分配方法
``` c++
// 分配不低于size_t x的空间, alloc_failmode分配失败策略, 默认退出并抛出oom
void* grow(size_t x, AllocFailType alloc_failmode = AllocFailStrategy::EXIT_OOM);
```

## ResourceArea
> hotspot/share/memory/resourceArea.hpp#ResourceArea

被ResourceMark使用, 继承Area, 一般情况下ResourceArea的内存标记为mtThread, 每一个线程分配储存

## ResourceMark
负责所有资源的分配与销毁, 在构造函数中配置ResourceArea, 并在析构函数中释放已分配的资源

### 配置
``` c++
//初始化方法, 不同的构造函数都会执行该方法
void initialize(Thread *thread) {
    _area = thread->resource_area();//使用线程中的ResourceArea
    _chunk = _area->_chunk;//指向当前操作的chunk, 释放时使用
    _hwm = _area->_hwm;//高水位
    _max= _area->_max;//最大值
    _size_in_bytes = _area->size_in_bytes();//已使用的空间,单位byte, 释放时使用
    debug_only(_area->_nesting++;)
    assert( _area->_nesting > 0, "must stack allocate RMs" );
#ifdef ASSERT//ASSERT编译使用
    _thread = thread;
    _previous_resource_mark = thread->current_resource_mark();
    thread->set_current_resource_mark(this);
#endif // ASSERT
```

### 释放资源
``` c++
//析沟函数调用
void reset_to_mark() {
    if (UseMallocOnly) free_malloced_objects();//如果使用malloc/free调用, 默认不启用

    if( _chunk->next() ) {//删除之后的chunk
      //设置构造时记录的size_in_bytes, 在删除前设置, 否则arena可能会超过chunk
      assert(_area->size_in_bytes() > size_in_bytes(), "Sanity check");
      _area->set_size_in_bytes(size_in_bytes());
      _chunk->next_chop();
    } else {//未扩展过chunk
      assert(_area->size_in_bytes() == size_in_bytes(), "Sanity check");
    }
    _area->_chunk = _chunk;//恢复到构造时的状态
    _area->_hwm = _hwm;
    _area->_max = _max;

    //将当前的chunk, 标记为0xAB, for debug
    if (ZapResourceArea) memset(_hwm, badResourceValue, _max - _hwm);
  }
```

## 内存类型
> hotspot/share/memory/allocation.hpp

Openjdk定义了内存类型, 方便内存追踪, 在allocation.hpp, 展开后以mt开头

``` c++

#define MEMORY_TYPES_DO(f) \
  /* Memory type by sub systems. It occupies lower byte. */  \
  f(mtJavaHeap,      "Java Heap")   /* Java heap                                 */ \
  f(mtClass,         "Class")       /* Java classes                              */ \
  f(mtThread,        "Thread")      /* thread objects                            */ \
  f(mtThreadStack,   "Thread Stack")                                                \
  f(mtCode,          "Code")        /* generated code                            */ \
  f(mtGC,            "GC")                                                          \
  f(mtCompiler,      "Compiler")                                                    \
  f(mtJVMCI,         "JVMCI")                                                       \
  f(mtInternal,      "Internal")    /* memory used by VM, but does not belong to */ \
                                    /* any of above categories, and not used by  */ \
                                    /* NMT                                       */ \
  f(mtOther,         "Other")       /* memory not used by VM                     */ \
  f(mtSymbol,        "Symbol")                                                      \
  f(mtNMT,           "Native Memory Tracking")  /* memory used by NMT            */ \
  f(mtClassShared,   "Shared class space")      /* class data sharing            */ \
  f(mtChunk,         "Arena Chunk") /* chunk that holds content of arenas        */ \
  f(mtTest,          "Test")        /* Test type for verifying NMT               */ \
  f(mtTracing,       "Tracing")                                                     \
  f(mtLogging,       "Logging")                                                     \
  f(mtStatistics,    "Statistics")                                                  \
  f(mtArguments,     "Arguments")                                                   \
  f(mtModule,        "Module")                                                      \
  f(mtSafepoint,     "Safepoint")                                                   \
  f(mtSynchronizer,  "Synchronization")                                             \
  f(mtNone,          "Unknown")                                                     \
  //end

#define MEMORY_TYPE_DECLARE_ENUM(type, human_readable) \
  type,

/*
 * Memory types
 */
enum MemoryType {
  MEMORY_TYPES_DO(MEMORY_TYPE_DECLARE_ENUM)
  mt_number_of_types   // number of memory types (mtDontTrack
                       // is not included as validate type)
};

typedef MemoryType MEMFLAGS;

```