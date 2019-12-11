# G1垃圾回收
g1将所有堆分割成一份份的heapRegion

## tag
每一份heapRegion根据tag区分该region的属性, 如是否是yound或old还是大堆.

每一个tag可以看作一个整型, 分为两个部分:
* 高位(N -1 bits)为主类型, 指定region的年代, 如yound, old, humongous, archive.
* 最低位为次要类型, 指定region细分作用, 如young中是eden还是survivor

所有类型表示如下:
```
  00000 0 [ 0] Free

  00001 0 [ 2] Young Mask
  00001 0 [ 2] Eden
  00001 1 [ 3] Survivor
  
  00010 0 [ 4] Humongous Mask
  00100 0 [ 8] Pinned Mask
  00110 0 [12] Starts Humongous
  00110 1 [13] Continues Humongous
  
  01000 0 [16] Old Mask
  
  10000 0 [32] Archive Mask
  11100 0 [56] Open Archive
  11100 1 [57] Closed Archive
```

## region大小
heapRegionBounds确定每一份region的边界, 默认值如下:
* MIN_REGION_SIZE = 1024 * 1024 //最小1M
* MAX_REGION_SIZE = 32 * 1024 * 1024//最大32M


