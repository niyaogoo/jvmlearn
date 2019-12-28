# 虚拟机互斥锁
* Thread::muxAcquire
* Thread::muxRelease

非公平锁, 每一个线程都尝试将自己park到lock

## ParkEvent
每一个线程对应一个ParkEvent,ParkEvent分配在CHeap, 无法被销毁

## LOCKBIT
LOCKBIT作为判断是否有锁以及队列的下一个ParkEvent的指针
* 0 无锁
* 1 有锁并且占用
* 最低位0 已释放但是可能有其他线程竞争该锁

## 获取锁
```c++
-> /hotspot/share/runtime/thread.cpp

//获取互斥锁
//使用LOCKBIT作为锁的最低位, 判断当前无锁或是有线程等待
const intptr_t LOCKBIT = 1;


void Thread::muxAcquire(volatile intptr_t * Lock, const char * LockName) {
  //尝试cas锁值为1
  intptr_t w = Atomic::cmpxchg(LOCKBIT, Lock, (intptr_t)0);
  //如果返回0说明cas成功当前无锁, 成功获取到锁, 此时Lock值为1
  if (w == 0) return;
  //如果返回值最低位为0说明锁已被释放但是可能会有其他线程竞争
  //尝试将刚刚获取到的w(其他线程存放的ParkEvent指针)最低位置1尝试存放
  if ((w & LOCKBIT) == 0 && Atomic::cmpxchg(w|LOCKBIT, Lock, w) == w) {
    //成功cas说明获取到锁
    return;
  }
  //如果失败则说明锁竞争失败, 使用本线程的ParkEvent
  ParkEvent * const Self = Thread::current()->_MuxEvent;
  //断言本线程的ParkEvent最低位为0
  assert((intptr_t(Self) & LOCKBIT) == 0, "invariant");
  //开始循环尝试获取锁或尝试park至Lock
  for (;;) {
    //spin次数, 如果是cpu是多核则100次，否则1次
    int its = (os::is_MP() ? 100 : 0) + 1;

    while (--its >= 0) {
      //w为Lock指向的值
      w = *Lock;
      //如果最低位为0说明仍然可以竞争
      if ((w & LOCKBIT) == 0 && Atomic::cmpxchg(w|LOCKBIT, Lock, w) == w) {
        return;
      }
    }

    //将ParkEvent中的_event设置为0
    Self->reset();
    //ParkEvent的OnList指向Lock
    Self->OnList = intptr_t(Lock);

    //非必要的屏障，cas本身是序列化执行
    OrderAccess::fence();
    for (;;) {
      //再次检查Lock指向的值
      w = *Lock;
      if ((w & LOCKBIT) == 0) {
        if (Atomic::cmpxchg(w|LOCKBIT, Lock, w) == w) {
          //同样的逻辑, 如果成功OnList注销
          Self->OnList = 0;
          return;
        }
        //失败重试spin或park
        continue;
      }
      //断言Lock指向的值最低位为1, 走到这一步肯定
      assert(w & LOCKBIT, "invariant");
      //准备park，将w最低位置为0并排在当前线程的ParkEvent之后
      //注意最低位为0
      Self->ListNext = (ParkEvent *) (w & ~LOCKBIT);
      //尝试park, 将本线程的ParkEvent地址最低位置1告诉其他线程已占用,尝试cas
      if (Atomic::cmpxchg(intptr_t(Self)|LOCKBIT, Lock, w) == w) break;
    }

    //执行park, 这个while用来被唤醒后可能的重新park
    while (Self->OnList != 0) {
      Self->park();
    }
  }
}
```

## 释放锁
```c++
-> /hotspot/share/runtime/thread.cpp

void Thread::muxRelease(volatile intptr_t * Lock)  {
  for (;;) {
    //cas尝试将Lock的值置为1
    const intptr_t w = Atomic::cmpxchg((intptr_t)0, Lock, LOCKBIT);
    //获取锁的情况下最低位肯定为1
    assert(w & LOCKBIT, "invariant");
    //如果Lock指向的值w为1说明无其他线程等待
    if (w == LOCKBIT) return;
    //将最低位置为0获取真正的ParkEvent
    ParkEvent * const List = (ParkEvent *) (w & ~LOCKBIT);
    //断言ParkEvent不为NULL
    assert(List != NULL, "invariant");
    //断言ParkEvent等待在当前Lock
    assert(List->OnList == intptr_t(Lock), "invariant");
    //获取当前阻塞中的线程unpark并将其下一个ParkEvent
    ParkEvent * const nxt = List->ListNext;
    guarantee((intptr_t(nxt) & LOCKBIT) == 0, "invariant");

    //将Lock指向的ParkEvent的下一个ParkEvent设置到Lock并准备unpark
    //有可能其他线程尝试park, 需要cas
    if (Atomic::cmpxchg(intptr_t(nxt), Lock, w) != w) {
      continue;
    }
    //将ParkEvent注销Lock
    List->OnList = 0;
    //内存屏障
    OrderAccess::fence();
    //unpark线程
    List->unpark();
    return;
  }
}
```