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

内存类型安全的Chunk池, 减少malloc/free的抖动, 未使用Mutex锁, 因为pool在线程初始化之前就已使用.

在Threads::create_vm()->vm_init_globals()->chunkpool_init()进行初始化.

默认4种类型, 对应Chunk中的tiny_size, init_size等, 分别有固定的size, 在分配Chunk时寻找对应的ChunkPool进行分配.
* _large_pool
* _medium_pool
* _small_pool
* _tiny_pool

### 定义
``` c++

class ChunkPool: public CHeapObj<mtInternal> {
  Chunk*       _first; //第一块chunk
  size_t       _num_chunks;   // 未使用的chunk数量
  size_t       _num_used;     // 已使用的chunk数量
  const size_t _size;         // 每一块chunk的大小, 每一个ChunkPool都固定
```

### 内存分配与释放
``` c++

  //获取第一块chunk
  void* get_first() {
    Chunk* c = _first;
    if (_first) {//注意如果没有调用free归还则first永远为NULL
      _first = _first->next();//first指向的是最后一次归还的Chunk
      _num_chunks--;//已使用-1
    }
    return c;
  }

  //分配方法入口, 分配一个新的chunk或者使用已有的
  NOINLINE void* allocate(size_t bytes, AllocFailType alloc_failmode) {
    //参数bytes必须固定, init_size, small_size等
    assert(bytes == _size, "bad size");
    void* p = NULL;
    //使用TC锁时无法获取虚拟机锁(Mutex), 所以调用os::malloc应该在TC锁外部
    { ThreadCritical tc;
      _num_used++;
      p = get_first();//see above
    }
    //无空闲chunk时进行os分配
    if (p == NULL) p = os::malloc(bytes, mtChunk, CURRENT_PC);
    if (p == NULL && alloc_failmode == AllocFailStrategy::EXIT_OOM) {
      vm_exit_out_of_memory(bytes, OOM_MALLOC_ERROR, "ChunkPool::allocate");
    }
    return p;
  }

  //释放chunk
  void free(Chunk* chunk) {
    assert(chunk->length() + Chunk::aligned_overhead_size() == _size, "bad size");
    //tc锁
    ThreadCritical tc;
    //已使用-1
    _num_used--;

    //归还chunk, pool->first指向归还的chunk
    chunk->set_next(_first);
    _first = chunk;
    _num_chunks++;//未使用+1, see get_first::_num_chunks--
  }
```

## Area
> hotspot/share/memory/area.hpp#Area

负责内存的快速分配, 管理Chunk
### 配置
``` c++
  MEMFLAGS    _flags;           //内存追踪标记

  Chunk *_first;                //第一个chunk
  Chunk *_chunk;                //当前chunk
  char *_hwm, *_max;            //指向当前chunk的bottom与top, _hwm指向当前空闲的内存单元, _max指向最大空闲堆顶
```

### 分配方法
``` c++
//arena.hpp中重载了3个Amalloc方法
void* Amalloc(size_t x, AllocFailType alloc_failmode = AllocFailStrategy::EXIT_OOM)

void *Amalloc_4(size_t x, AllocFailType alloc_failmode = AllocFailStrategy::EXIT_OOM)

void* Amalloc_D(size_t x, AllocFailType alloc_failmode = AllocFailStrategy::EXIT_OOM)

//Amalloc_4与Amalloc_D效果一样, 分配指定长度的内存, 必须与char*对齐, 在Symbol中使用
//ResourceArea默认使用Amalloc

void* Amalloc(size_t x, AllocFailType alloc_failmode = AllocFailStrategy::EXIT_OOM) {
  //ARENA_AMMLOC_ALIGNMENT必须2的幂次方, LP64为16字节
  assert(is_power_of_2(ARENA_AMALLOC_ALIGNMENT) , "should be a power of 2");
  //与ARENA_AMMLOC_ALIGHMENT对齐
  x = ARENA_ALIGN(x);
  debug_only(if (UseMallocOnly) return malloc(x);)
  if (!check_for_overflow(x, "Arena::Amalloc", alloc_failmode))
    return NULL;
  //当前chunk无空闲分配新的chunk
  if (_hwm + x > _max) {
    return grow(x, alloc_failmode);
  } else {
    //在当前chunk分配
    char *old = _hwm;
    _hwm += x;
    return old;
  }
}

// area无空闲chunk时分配新的chunk
// 分配不低于size_t x的空间, alloc_failmode分配失败策略, 默认退出并抛出oom
void* Arena::grow(size_t x, AllocFailType alloc_failmode) {
  // Get minimal required size.  Either real big, or even bigger for giant objs
  size_t len = MAX2(x, (size_t) Chunk::size);

  Chunk *k = _chunk;//获取当前chunk的地址
  //尝试将当前_chunk指向新分配的chunk
  _chunk = new (alloc_failmode, len) Chunk(len);
  
  if (_chunk == NULL) {
    //如果chunk分配失败重置_chunk
    _chunk = k;
    return NULL;
  }
  //如果老的_chunk不为NULL,将原有的chunk->next指向新的chunk
  if (k) k->set_next(_chunk);
  //否则_first与_chunk一样指向新分配的chunk
  else _first = _chunk;
  //将area中的_hwm与_max设置为新分配chunk的bottom与top
  _hwm  = _chunk->bottom();
  _max =  _chunk->top();
  //设置area的长度
  set_size_in_bytes(size_in_bytes() + len);
  //分配完chunk后偏移x,让出第一块需要的内存并返回
  void* result = _hwm;
  _hwm += x;
  return result;
}
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
    _hwm = _area->_hwm;//area管理的chunk的bottom
    _max= _area->_max;//area管理的chunk的max
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

## ThreadCritical
> hotspot/share/runtime/threadCritical.hpp

在虚拟机初始化完毕之前使用该锁, 全局唯一

``` c++
//linux tc实现
//hotspot/os/linux/threadCritical_linux.cpp

static pthread_t             tc_owner = 0;//锁拥有者
static pthread_mutex_t       tc_mutex = PTHREAD_MUTEX_INITIALIZER;//锁
static int                   tc_count = 0;//可重入计数器

//自动构造
ThreadCritical::ThreadCritical() {
  pthread_t self = pthread_self();
  //如果当前不是自己则尝试获取
  if (self != tc_owner) {
    int ret = pthread_mutex_lock(&tc_mutex);
    guarantee(ret == 0, "fatal error with pthread_mutex_lock()");
    assert(tc_count == 0, "Lock acquired with illegal reentry count.");
    tc_owner = self;
  }
  //已获得锁, 重入+1
  tc_count++;
}

//自动析构
ThreadCritical::~ThreadCritical() {
  assert(tc_owner == pthread_self(), "must have correct owner");
  assert(tc_count > 0, "must have correct count");

  //重入-1
  tc_count--;
  //重入归零后释放锁
  if (tc_count == 0) {
    tc_owner = 0;
    int ret = pthread_mutex_unlock(&tc_mutex);
    guarantee(ret == 0, "fatal error with pthread_mutex_unlock()");
  }
}

```