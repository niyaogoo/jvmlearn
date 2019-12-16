# universe

虚拟机堆、元数据空间、线程等分配都在Universe::universe_init()中完成.
> hotspot/share/memory/universe.cpp

## hard code offset

## global behaviours

## init heap
初始化虚拟机堆
```c++
GCConfig::arguments()->initialize_heap_sizes();//初始化虚拟机堆参数

  //gcArguments.cpp
  void GCArguments::initialize_heap_sizes() {
    initialize_alignments();// 对于g1来说设置region的大小与对齐等相关参数
    initialize_heap_flags_and_sizes();// 初始化堆相关虚拟机flag并设置对齐
    initialize_size_info();// 检查各堆参数是否正确
  }

jint status = Universe::initialize_heap();

//.. 
jint Universe::initialize_heap() {
  assert(_collectedHeap == NULL, "Heap already created");
  _collectedHeap = GCConfig::arguments()->create_heap();// 对于g1来说只是new G1CollectedHeap()
  jint status = _collectedHeap->initialize();// 这一步初始化堆, see g1gc.md

  if (status == JNI_OK) {
    log_info(gc)("Using %s", _collectedHeap->name());
  }

  return status;
}
```
## tlab
Thread local allocation buffer,线程本地分配缓冲。
当需要在堆上分配对象时, 如果TLAB开启, 对象首先在TLAB中分配, 这一部分只存在于edankk空间中，每个线程有自己的TLAB, 这样在分配内存时就无需加锁。
> -XX:+UseTLAB 是否使用, -XX:TLABSize 大小, -XX:+ResizeTLAB 允许动态调整tlab
```c++
Universe::initialize_tlab();

void Universe::initialize_tlab() {
  ThreadLocalAllocBuffer::set_max_size(Universe::heap()->max_tlab_size());//在g1gc中tlab最大为大文件下限, 64bit上为64kb
  if (UseTLAB) {
    assert(Universe::heap()->supports_tlab_allocation(),
           "Should support thread-local allocation buffers");
    ThreadLocalAllocBuffer::startup_initialization();
  }
}

void ThreadLocalAllocBuffer::startup_initialization() {
  ThreadLocalAllocStats::initialize();//初始化统计信息
  _target_refills = 100 / (2 * TLABWasteTargetPercent);
  _target_refills = MAX2(_target_refills, 2U);
  //默认在达到50%时进行垃圾回收
  //如果开启C2编译器, 则需要更多的空间:_reserve_for_allocation_prefetch
  Thread::current()->tlab().initialize();//初始化当前线程的tlab
  //..
}
```
## metaspace
初始化元空间
如果开启CDS将共享文档空间映射至当前虚拟机内存中, 如果未开启或未找到归档空间则使用class space, 在虚拟机堆结束地址上分配元空间.

```c++
Metaspace::global_initialize();
//..
char* base = (char*)align_up(CompressedOops::end(), _reserve_alignment);
      allocate_metaspace_compressed_klass_ptrs(base, 0); 
//..
```
> CDS: class data sharing, 将文件映射至共享内存, 减少虚拟机加载时间;
