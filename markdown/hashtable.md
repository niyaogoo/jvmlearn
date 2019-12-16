# Hotspot中的hashtable
> src/hotspot/share/utilities/hashtable.hpp
hotspot中的hashtable, 以泛型实现, 用来存放符号表与字符串表

## BasicHashtableEntry
基础Hashtable入口, 计算32位hash值, 链表形式存放在bucket中, _next指针指向下一个entry,
由于地址最小与2对齐,_next最低位指定entry是否共享, 1表示共享, 0表示不共享.
```c++
  //make_ptr屏蔽最低位
  static BasicHashtableEntry<F>* make_ptr(BasicHashtableEntry<F>* p) {
    return (BasicHashtableEntry*)((intptr_t)p & -2);
  }
```

## BasicHashtable
基础hash表, Hashtable的父类
### 构造参数
* table_size: 初始容量
* entry_size: 每一个entry的大小
* number_of_entries: entry数量

### 字段说明
block是指每次扩容分配的一块内存区
* _free_list: 指向当前可用的entry
* _first_free_entry: 指向最后一次扩容后的第一个可用entry, 每次添加entry自动指向下一个可用entry地址
* _entry_blocks: 可增长数组, 储存每次扩容的block的第一个entry
* _end_block: 指向最后一次扩容的最后一个entry

### 扩容策略
block_size = _table_size的一半或_number_of_entries两者的最大值, 不能超过512
len取block_size * _entry_size的向下最大2次方
```c++
template <MEMFLAGS F> BasicHashtableEntry<F>* BasicHashtable<F>::new_entry(unsigned int hashValue) {
  BasicHashtableEntry<F>* entry = new_entry_free_list();

  if (entry == NULL) {
    if (_first_free_entry + _entry_size >= _end_block) {
      int block_size = MIN2(512, MAX2((int)_table_size / 2, (int)_number_of_entries));
      int len = _entry_size * block_size;
      len = 1 << log2_int(len); // round down to power of 2
      assert(len >= _entry_size, "");
      _first_free_entry = NEW_C_HEAP_ARRAY2(char, len, F, CURRENT_PC);
      _entry_blocks->append(_first_free_entry);
      _end_block = _first_free_entry + len;
    }
    entry = (BasicHashtableEntry<F>*)_first_free_entry;
    _first_free_entry += _entry_size;
  }

  assert(_entry_size % HeapWordSize == 0, "");
  entry->set_hash(hashValue);
  return entry;
}
```

### 添加策略
1. 分配一个entry指针, 指向_first_free_entry的第一个地址
2. 计算key的hash, 并与_table_size模运算获取bucket的位置
3. 将bucket原来指向的entry作为新entry的next
4. 增加number_of_entries
```c++
void add(K key, V value) {
    unsigned int hash = HASH(key);
    KVHashtableEntry* entry = new_entry(hash, key, value);
    BasicHashtable<F>::add_entry(BasicHashtable<F>::hash_to_index(hash), entry);
}
```

### 默认hash算法
使用地址计算hash, 右移3位异或, 防止内存对齐落在同一个bucket
```c++
template<typename K> unsigned primitive_hash(const K& k) {
  unsigned hash = (unsigned)((uintptr_t)k);
  return hash ^ (hash >> 3);
}
```

## Hashtable

## KVHashtable
最简单的hashtable实现
