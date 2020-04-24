# synchronized虚拟机实现

hotspot对synchronized实现依赖对象头(markword)与平台互斥锁, 对不同的状态采取不同的锁策略

### 锁策略有以下

* 使用偏向锁(biase), 偏向一个java线程, 该线程无需原子操作即可上锁与解锁. 偏向锁.
* 使用cas形式对mark上锁, 将栈帧顶部的BasicObjectMonitor地址cas至mark, 完成上锁. 轻锁.
* 使用spin与互斥锁(mutex)形式设置或挂起当前线程. 重锁.


markword用于描述对象头, 使用低2位描述对象锁的状态, 低底3位描述是否偏向, 在64bit形式下有以下形式:

### 偏向锁

>  [JavaThread* | epoch | age | 1 | 01]       对象锁偏向JavaThread

>  [0           | epoch | age | 1 | 01]       匿名偏向

### 轻锁
>    [BasicObjectMonitor地址]

### 重锁

>    [ptr             | 00]  锁定             已被ObjectMonitor锁定

>    [header      | 0 | 01]  未锁             普通状态

>    [ptr             | 10]  膨胀到重锁        已膨胀到重锁, header被替换

>    [ptr             | 11]  marked           标记位移除


## 入口
入口代码, 同时获取需要同步的oop对象
>hotspot/cpu/x86/templateInterpreterGenerator_x86.cpp#TemplateInterpreterGenerator::generate_normal_entry

该函数生成运行时代码入口点, 参数synchronized确定该方法是否需要加锁;

代码中的__为
> #define __ Disassembler::hook<InterpreterMacroAssembler>(__FILE__, __LINE__, _masm)->

_masm为MacroAssembler, 用于生成机器指令.

``` c++
//hotspot/cpu/x86/templateInterpreterGenerator_x86.cpp


address TemplateInterpreterGenerator::generate_normal_entry(bool synchronized) {
  //..

  //oop锁入口点
  if (synchronized) {
    lock_method();
  }
}

//锁操作入口点, rbx寄存器指向Method指针, r14寄存器指向本地变量表
void TemplateInterpreterGenerator::lock_method() {
  //注意rbx寄存器在之前(generate_normal_entry)时已被指向Method指针
  // synchronize method
  //access flags地址
  const Address access_flags(rbx, Method::access_flags_offset());
  //rbp寄存器指向栈底, interpreter_frame_monitor_block_top_offset指向初始化栈帧顶部, 用于BasicObjectLock分配
  //jvm调用栈简单说明
  //
  //    [expression stack      ] * <- sp栈顶
  //    [monitors              ]   \ 
  //     ...                        | BasicObjectLock 块
  //    [monitors              ]   /
  //    [monitor block size    ]
  //    [byte code pointer     ]                   = 字节码指针()                bcp_offset
  //    [pointer to locals     ]                   = 本地变量表()             locals_offset
  //    [constant pool cache   ]                   = 常量池()              cache_offset
  //    [methodData            ]                   = MethodData()                mdx_offset
  //    [Method*               ]                   = Method()             method_offset
  //    [last sp               ]                   = 调用者栈顶()            last_sp_offset
  //    [old stack pointer     ]                     (调用者栈顶)          sender_sp_offset
  //    [old frame pointer     ]   <- fp           = link()
  //    [return pc             ]                      函数返回地址
  //    [oop temp              ]                     (only for native calls)
  //    [locals and parameters ]                   本地变量与参数
  const Address monitor_block_top(
        rbp,
        frame::interpreter_frame_monitor_block_top_offset * wordSize);
  //BasicObjectLock大小
  const int entry_size = frame::interpreter_frame_monitor_size() * wordSize;

  //获取要同步的对象
  {
    Label done;//static跳转偏移量
    __ movl(rax, access_flags);
    __ testl(rax, JVM_ACC_STATIC);//判断是否是static, 如果有static标记则ZF标记为1, 需要获取oop对象
    // get receiver (assume this is frequent case)
    //rlocals寄存器在64bit为r14寄存器, 指向本地变量偏移0, 将rax寄存器指向同步对象
    __ movptr(rax, Address(rlocals, Interpreter::local_offset_in_bytes(0)));
    //判断ZF标记位, static对象无需获取实际oop对象
    __ jcc(Assembler::zero, done);
    //非static对象获取oop对象
    __ load_mirror(rax, rbx);

    //绑定非static偏移跳转
    __ bind(done);
    //rax不能为NULL
    __ resolve(IS_NOT_NULL, rax);
  }

  //增加栈空间, 一个BasicObjectLock长度
  __ subptr(rsp, entry_size);
  //将monitor_block_top(BasicObjectLock)指向栈顶
  __ movptr(monitor_block_top, rsp);  // set new monitor block top
  //将BasicObjectLock的obj指向需要同步的oop对象
  __ movptr(Address(rsp, BasicObjectLock::obj_offset_in_bytes()), rax);
  //amd64使用rdx作为BasicObjectLock寄存器, 供之后使用
  const Register lockreg = NOT_LP64(rdx) LP64_ONLY(c_rarg1);
  //lockreg寄存器指向刚刚分配的BasicObjectLock
  __ movptr(lockreg, rsp); // object address
  //开始加锁
  __ lock_object(lockreg);

}
```

## 轻锁

> hotspot/cpu/x86/interp_masm_x86.cpp


``` c++

//开始加锁, lock_reg指向BasicObjectLock
void InterpreterMacroAssembler::lock_object(Register lock_reg) {
  //全局变量, 是否跳过偏向锁, 默认false
  if (UseHeavyMonitors) {
    //直接进入重锁在偏向锁失败后会再次使用, 具体在下方详解
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);
  } else {
    //偏向锁成功后指令偏移量
    Label done;
    //指定一些偏向锁需要的寄存器, swap_reg需要用rax寄存器以支持cmpxchg指令
    const Register swap_reg = rax;
    //交换时使用的中间寄存器
    const Register tmp_reg = rbx;
    //存放oop, amd64使用c_rarg3                              
    const Register obj_reg = LP64_ONLY(c_rarg3) NOT_LP64(rcx);

    //BasicObjectLock的一些偏移量
    const int obj_offset = BasicObjectLock::obj_offset_in_bytes();
    const int lock_offset = BasicObjectLock::lock_offset_in_bytes ();
    //注意mark_offset不直接在BasicObjectLock, 而是在BasicObjectLock中的BasicLock对象中
    const int mark_offset = lock_offset +
                            BasicLock::displaced_header_offset_in_bytes();

    //偏向锁失败后指令偏移量, 进入轻量级锁
    Label slow_case;

    //加载对象锁到obj_reg寄存器
    movptr(obj_reg, Address(lock_reg, obj_offset));

    //全局变量, 是否使用偏向锁
    if (UseBiasedLocking) {
      //偏向锁入口, see ## 偏向锁
      biased_locking_enter(lock_reg, obj_reg, swap_reg, tmp_reg, false, done, &slow_case);
    }

    //使用轻锁
    // swap = 1
    movl(swap_reg, (int32_t)1);

    // swap = object->mark() | 1, 假设对象未锁
    orptr(swap_reg, Address(obj_reg, oopDesc::mark_offset_in_bytes()));

    //保存可能被替换的mark到lock的mark
    movptr(Address(lock_reg, mark_offset), swap_reg);

    //对cmpxchgptr加锁直接读取内存
    lock();
    //比较swap(rax)与oop的markword, 一致说明cas成功
    cmpxchgptr(lock_reg, Address(obj_reg, oopDesc::mark_offset_in_bytes()));
    jcc(Assembler::zero, done);

    
    const int zero_bits = LP64_ONLY(7) NOT_LP64(3);

    //到这说明失败, swap(rax)在上cmpxchgptr失败后为oop的mark值
    //将检查是否是本线程上的锁(重入)
    //  1) (mark & zero_bits) == 0, and
    //  2) rsp <= mark < mark + os::pagesize()
    // 可演化为以下表达式
    subptr(swap_reg, rsp);
    andptr(swap_reg, zero_bits - os::vm_page_size());

    movptr(Address(lock_reg, mark_offset), swap_reg);
    jcc(Assembler::zero, done);

    //到这说明cas失败, 进入重锁
    bind(slow_case);

    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);

    bind(done);
  }
}
```

## 偏向锁实现

> macroAssembler_x86.cpp

``` c++

int MacroAssembler::biased_locking_enter(Register lock_reg,
                                         Register obj_reg,
                                         Register swap_reg,
                                         Register tmp_reg,
                                         bool swap_reg_contains_mark,
                                         Label& done,
                                         Label* slow_case,
                                         BiasedLockingCounters* counters) {
  //BasicObjectLock中的_obj对象, 注意是获取到锁的对象
  Address mark_addr      (obj_reg, oopDesc::mark_offset_in_bytes());

  // Biased locking
  // See whether the lock is currently biased toward our thread and
  // whether the epoch is still valid
  // Note that the runtime guarantees sufficient alignment of JavaThread
  // pointers to allow age to be placed into low bits
  // First check to see whether biasing is even enabled for this object
  //确认当前偏向锁是否是当前线程, 并且对象年代还能支持偏向锁
  Label cas_label;
  int null_check_offset = -1;
  //swap寄存器中是否包含了mark, 如果没有则从obj_reg中获取
  if (!swap_reg_contains_mark) {
    null_check_offset = offset();
    movptr(swap_reg, mark_addr);
  }
  //判断当前markword是否支持偏向锁, 低三位101, 说明是有偏向锁并未膨胀
  movptr(tmp_reg, swap_reg);
  andptr(tmp_reg, markWord::biased_lock_mask_in_place);
  cmpptr(tmp_reg, markWord::biased_lock_pattern);
  //如果非101, 则直接跳过, 不使用偏向锁, 尝试清锁
  jcc(Assembler::notEqual, cas_label);

  //指令执行到这说明支持偏向锁, 并未锁
  if (swap_reg_contains_mark) {
    null_check_offset = offset();
  }
  //加载klass中的markword
  load_prototype_header(tmp_reg, obj_reg);

  //tmp_reg指向klass的markword(prototype_header), r15_thread指向当前线程对象, 这一步结果Thread*---101
  orptr(tmp_reg, r15_thread);
  //swap_reg为当前线程栈中指向的oop的markword, 异或操作, 这一步如果当前线程占用锁则为000---000
  xorptr(tmp_reg, swap_reg);
  Register header_reg = tmp_reg;
  //age_mask_in_place: ---0001111000, 去掉低三位的1bit偏向位与2bit锁位, 取反与操作, 1111--0000111, 如果是当前线程占用则000---000
  andptr(header_reg, ~((int) markWord::age_mask_in_place));
  //zf标记位0跳转done(参数传入地址), 也就是andptr指令结果为0
  jcc(Assembler::equal, done);


  //下方代码开始尝试获取偏向锁
  //对象支持偏向锁但是不是当前线程拥有, 开始尝试获取对象头进行相关操作
  Label try_revoke_bias;
  Label try_rebias;

  //注意如果低三位在上方andptr指令执行后未被清除, 则说明对象不支持偏向锁, 取消本次偏向锁执行
  testptr(header_reg, markWord::biased_lock_mask_in_place);
  jccb(Assembler::notZero, try_revoke_bias);

  //到这说明对象任然支持偏向锁, 检查偏向对象的年代位是否有效, 也就是markword的年代位与klass的prototype_header一致
  //注意对象的年代位只有在safepoint时才会改变, 如果markword与prototype_header的年代位不一致, 会从新获取prototype_header并重偏向到??
  //epoch掩码为0011 0000 0000
  testptr(header_reg, markWord::epoch_mask_in_place);
  jccb(Assembler::notZero, try_rebias);//rebias

  //到这说明年代一致, 检查偏向锁所有者, 可能被设置, 也可能被清除, 尝试
  //用原子操作(compare and exchange)获取锁拥有者, 如果失败则退出获取偏向锁
  //第一次构造一个未偏向的markword, 防止意外的影响其他线程的正常偏向操作
  //swap_reg指向同步对象的markword 
  
  //构造一个未偏向的markword, 指向当前线程
  andptr(swap_reg,
         markWord::biased_lock_mask_in_place | markWord::age_mask_in_place | markWord::epoch_mask_in_place);
  movptr(tmp_reg, swap_reg);
  //准备替换的markword, thread指向当前线程
  orptr(tmp_reg, r15_thread);

  //对cmpxchgptr增加lock前缀获取内存实际值
  lock();
  //尝试替换
  //如果偏向失败, 则当前线程放弃偏向, 跳转至cas锁
  cmpxchgptr(tmp_reg, mark_addr);//成功zf位为0
  
  if (slow_case != NULL) {
    //上一指令结果不为0, 则尝试
    //zf位为1, cmpxchgptr失败 跳转到cas
    jcc(Assembler::notZero, *slow_case);
  }
  //偏向成功
  jmp(done);

  bind(try_rebias);
  //跳转到这说明年代位已过期, 偏向锁拥有者已无效, 只有在这种情况下才能用当前对象头进行cas, 换句话说偏向锁可以切换线程
  
  load_prototype_header(tmp_reg, obj_reg);
  orptr(tmp_reg, r15_thread);
  lock();
  //与上方逻辑一致
  cmpxchgptr(tmp_reg, mark_addr);
  
  if (slow_case != NULL) {
    jcc(Assembler::notZero, *slow_case);
  }
  jmp(done);

  bind(try_revoke_bias);
  //执行到这说明对象已再支持偏向, 尝试重置对象的prototype_header, 去除偏向位
  load_prototype_header(tmp_reg, obj_reg);
  lock();
  cmpxchgptr(tmp_reg, mark_addr);

  bind(cas_label);

  return null_check_offset;
}
```

## 重锁

当轻锁失败后, 膨胀到重锁

``` c++
// hotspot/share/interpreter/interpreterRuntime.cpp

//重锁入口

JRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))

  Handle h_obj(thread, elem->obj());
  ObjectSynchronizer::enter(h_obj, elem->lock(), CHECK);

JRT_END

// hotspot/share/runtime/synchronizer.cpp

void ObjectSynchronizer::enter(Handle obj, BasicLock* lock, TRAPS) {
  //全局变量, 是否使用偏向
  if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::revoke(obj, THREAD);
    } else {
      BiasedLocking::revoke_at_safepoint(obj);
    }
  }

  markWord mark = obj->mark();
  assert(!mark.has_bias_pattern(), "should not see bias pattern here");

  if (mark.is_neutral()) {
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    lock->set_displaced_header(mark);
    if (mark == obj()->cas_set_mark(markWord::from_pointer(lock), mark)) {
      return;
    }
    // Fall through to inflate() ...
  } else if (mark.has_locker() &&
             THREAD->is_lock_owned((address)mark.locker())) {
    assert(lock != mark.locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark().value(), "don't relock with same BasicLock");
    lock->set_displaced_header(markWord::from_pointer(NULL));
    return;
  }

  // The object header will never be displaced to this lock,
  // so it does not matter what the value is, except that it
  // must be non-zero to avoid looking like a re-entrant lock,
  // and must not look locked either.
  lock->set_displaced_header(markWord::unused_mark());
  inflate(THREAD, obj(), inflate_cause_monitor_enter)->enter(THREAD);
}
```

## 附录
``` c++
//分配在栈顶的BasicObjectLock对象
class BasicObjectLock {
  friend class VMStructs;
 private:
  // 实际lock对象, 双字对齐
  BasicLock _lock;
  //获取锁的对象
  oop       _obj;
  //..
}
//在BasicObjectLock中使用的BasicLock, 
class BasicLock {
 private:
  //被替换的对象头
  volatile markWord _displaced_header;
}
```