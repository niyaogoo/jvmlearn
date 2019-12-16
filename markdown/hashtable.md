# Hotspot中的hashtable
> src/hotspot/share/utilities/hashtable.hpp
hotspot中的hashtable, 以泛型实现, 用来存放符号表与字符串表

## BasicHashtableEntry
基础Hashtable入口, 计算32位hash值, 链表形式存放._next指针指向下一个entry,
_next最低位指定entry是否共享, 1表示共享, 0表示不共享. 
```c++
  //make_ptr屏蔽最低位
  static BasicHashtableEntry<F>* make_ptr(BasicHashtableEntry<F>* p) {
    return (BasicHashtableEntry*)((intptr_t)p & -2);
  }
```

## HashtableEntry
继承BasicHashtableEntry, 维护字面量_literal

## HashtableBucket
entry所在的槽位(桶), 默认位hash % _table_size
AS
## B