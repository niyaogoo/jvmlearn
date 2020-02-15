# Metaspace 元空间

## 分配过程

* Metachunk为Metaspace分配内存的返回类型,其中的_top指针指向内存启示地址, _chunk_type说明该内存的类型(大小);
* Metaspace内部维护两个静态ChunkManager指针变量:_check_manager_metadata, _check_manager_class;
* ChunkManager内部维护类型为ChunkList(typedef class FreeList<Metachunk>)的_free_chunks, 保存空闲的可用chunk, 注意只保存specialized, small, medium类型(大小), Humongous保存在ChunkTreeDictionary中;
* 首先从_free_list或_humongous_dictionary中查找是否有大小匹配的内存进行分配;
* 在查找_free_list时会如果未找打指定大小的空闲内存,则尝试在更大的一块_free_list中分割成指定大小的空闲内存;
* 如果无空闲内存, 则在ChunkManager中维护的VistualSpaceList中分配内存;
* VistualSpaceList中维护的VistualSpaceNode(变量_current_vistual_space)中尝试分配内存;
* 首先计算当前VistualSpaceNode是否有足够的空间进行分配;
* 如果当前VistualSpaceNode有足够空间, 检查当前内存指针top是否与待分配内存对齐, 如果未对齐则分配填充内存, 并将填充内存存放入ChunkManager的_free_list中;
* 填充完毕后继续在VistualSpaceNode中分配内存


## Metabase
> /hotspot/share/memory/metaspace/metabase.hpp

Metabase为模板类, 是Metablock与Metachunk的超类, 存放在FreeList与BinaryTreeDictionary, 本身为双向链表.

``` c++
template <class T>
class Metabase {
  size_t _word_size;//占用字长
  T*     _next;下一个Metabase
  T*     _prev;上一个Metabase

```

## Metachunk
> /hotspot/share/memory/metaspace/metachunk.hpp
  /hotspot/share/memory/metaspace/metachunk.hpp

Metachunk继承Metabase, 记录所属VirtualSpaceNode, 数据实际地址等

``` c++
class Metachunk : public Metabase<Metachunk> {

  friend class ::MetachunkTest;

  //所属的VistualSpaceNode
  // The VirtualSpaceNode containing this chunk.
  VirtualSpaceNode* const _container;

  //分配时使用的地址, 由VirtualSpaceNode分配
  // Current allocation top.
  MetaWord* _top;

  //初始化时设置用于debug
  // A 32bit sentinel for debugging purposes.
  enum { CHUNK_SENTINEL = 0x4d4554EF,  // "MET"
         CHUNK_SENTINEL_INVALID = 0xFEEEEEEF
  };

  uint32_t _sentinel;

  //chunk类型
  const ChunkIndex _chunk_type;
  //是否是class, 将影响到对齐策略
  const bool _is_class;

  //记录该chunk是否空闲
  // Whether the chunk is free (in freelist) or in use by some class loader.
  bool _is_tagged_free;

  //chunk创建起源
  ChunkOrigin _origin;

  int _use_count;

```


## ChunkIndex ChunkSize

> /hotspot/share/memory/metaspace/metaspaceCommon.hpp

每一个ChunkIndex对应一个ChunkSize, specialized < small < Medium, 上一级能被下一级整除

``` c++
enum ChunkIndex {
  ZeroIndex = 0,
  SpecializedIndex = ZeroIndex, //对应128
  SmallIndex = SpecializedIndex + 1, //对应256 | 512
  MediumIndex = SmallIndex + 1, //对应4K | 8K
  HumongousIndex = MediumIndex + 1,//
  NumberOfFreeLists = 3,
  NumberOfInUseLists = 4
};

enum ChunkSizes {    // in words.
  ClassSpecializedChunk = 128,
  SpecializedChunk = 128,
  ClassSmallChunk = 256,
  SmallChunk = 512,
  ClassMediumChunk = 4 * K,
  MediumChunk = 8 * K
};

```

## OccupancyMap

> /hotspot/share/memory/metaspace/occupancyMap.hpp

OccupancyMap以bit形式记录VirtualSpaceNode的对应chunk头开始位置与是否使用

``` c++
class OccupancyMap : public CHeapObj<mtInternal> {

  //对应VirtualSpaceNode的low地址, 起始地址
  const MetaWord* const _reference_address;
  //对应VirtualSpaceNode的字长长度
  const size_t _word_size;

  //最小分块字长, 一般为special, 128字长
  const size_t _smallest_chunk_word_size;

  //核心字段, 2长度(layer)的1字节数组指针
  //layer1对应chunk的开始位置
  //layer2对应chunk的使用标记
  //所有layer储存1byte(8bit), 每一位bit对应VirtualSpaceNode的开始投, 
  //由于_smallest_chunk_word_size为最小块字长(128), 实际可能以small(256), medium(4k)对齐分配, 所以不一定所有bit都会为1
  //详细见构造函数与set_bit*函数
  uint8_t* _map[2];

  //对应_map[1|2]
  enum { layer_chunk_start_map = 0, layer_in_use_map = 1 };

  //_map总长度, _word_size / _smallest_chunk_word_size / 8
  size_t _map_size;

  //构造函数
  OccupancyMap::OccupancyMap(const MetaWord* reference_address, size_t word_size, size_t smallest_chunk_word_size) :
            _reference_address(reference_address), _word_size(word_size),
            _smallest_chunk_word_size(smallest_chunk_word_size) {
    //word_size对应virtualSpaceNode总字长
    //计算一共可以分为几块:num_bits
    size_t num_bits = word_size / smallest_chunk_word_size;
    //每一个byte可以标记8bit, 所以除8, +7为了防止出现_map_size为0
    _map_size = (num_bits + 7) / 8;
    _map[0] = (uint8_t*) os::malloc(_map_size, mtInternal);
    _map[1] = (uint8_t*) os::malloc(_map_size, mtInternal);
    memset(_map[1], 0, _map_size);
    memset(_map[0], 0, _map_size);
    
    //以下两个断言放在这可以加深理解原理, 开始地址与结束地址-smallest_chunk_word_size对应num_bit的第一个与最后一个对应chunk
    assert(get_bitpos_for_address(reference_address) == 0,
        "First chunk address in range must map to fist bit in bitmap.");
    assert(get_bitpos_for_address(reference_address + word_size - smallest_chunk_word_size) == num_bits - 1,
        "Last chunk address in range must map to last bit in bitmap.");
  }

  //给定一个地址, 计算对应的bit位
  unsigned get_bitpos_for_address(const MetaWord* p) const {
      const ptrdiff_t d = (p - _reference_address) / _smallest_chunk_word_size;
      return (unsigned) d;
  }

  //给定一个地址, 标记_map[0]对应的bit, v=true标记1, v=false标记0
  void set_chunk_starts_at_address(MetaWord* p, bool v) {
    //计算对应的bit位
    const unsigned pos = get_bitpos_for_address(p);
    //设置chunk起始位置标记为
    set_bit_at_position(pos, layer_chunk_start_map, v);
  }

  //修改对应layer的标记位
  void set_bit_at_position(unsigned pos, unsigned layer, bool v) {
    //pos为理论bit位, 实际储存以1byte(可描述8个pos位), 再次/8的到实际地址
    const unsigned byteoffset = pos / 8;
    //实际储存byte对应的bit位
    const unsigned mask = 1 << (pos % 8);
    //置1或0
    if (v) {
      //按位或标记1
      _map[layer][byteoffset] |= mask;
    } else {
      //按位取反标记0
      _map[layer][byteoffset] &= ~mask;
    }
  }
```