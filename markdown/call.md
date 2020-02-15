# 方法调用

## 从C调用Java方法

``` c++
// hotspot/share/runtime/javaCalls.cpp

//该函数为c调用java方法入口
//参数名对应返回值、java方法引用、调用参数、调用线程
void JavaCalls::call(JavaValue* result, const methodHandle& method, JavaCallArguments* args, TRAPS) {
  //断言调用线程为一个java线程
  assert(THREAD->is_Java_thread(), "only JavaThreads can make JavaCalls");
  // Need to wrap each and every time, since there might be native code down the
  // stack that has installed its own exception handlers
  //各操作实现自己的异常处理, 主要调用对象为call, linux的os_exception_wrapper仅是一个hook, 无其他操作, 直接调用call_helper
  //call_helper为函数指针, 负责执行调用  
  os::os_exception_wrapper(call_helper, result, method, args, THREAD);
  //请继续call_helper(..)
}

//主要调用函数
void JavaCalls::call_helper(JavaValue* result, const methodHandle& method, JavaCallArguments* args, TRAPS) {
  //..

  //桩代码, x86在stubGenerator_x86_64.cpp中初始化, 生成调用代码执行java方法
  StubRoutines::call_stub()(
          (address)&link,
          // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
          result_val_address,          // see NOTE above (compiler problem)
          result_type,
          method(),
          entry_point,
          parameter_address,
          args->size_of_parameters(),
          CHECK
        );
  //..
}

// hotspot/cpu/x86/stubGenerator_x86_64.cpp

  //桩代码要生成的栈结构
  // 只看Linux栈结构
  // Linux Arguments:
  //    c_rarg0:   call wrapper address                   address
  //    c_rarg1:   result                                 address
  //    c_rarg2:   result type                            BasicType
  //    c_rarg3:   method                                 Method*
  //    c_rarg4:   (interpreter) entry point              address
  //    c_rarg5:   parameters                             intptr_t*
  //    16(rbp): parameter size (in words)              int
  //    24(rbp): thread                                 Thread*
  //    
  //    以上为各个参数所在地址, 前6个参数按照约定存放在c_rarg0~6寄存器中(别名), 后续都存放在栈中
  //    参考call_helper中, parameter size与thread属于栈中参数
  //    以下为桩代码要生成的栈结构
  //     [ return_from_Java     ] <--- rsp 栈顶
  //     [ argument word n      ]
  //      ...
  // -12 [ argument word 1      ]
  // -11 [ saved r15            ] <--- rsp_after_call
  // -10 [ saved r14            ]
  //  -9 [ saved r13            ]
  //  -8 [ saved r12            ]
  //  -7 [ saved rbx            ]
  //  -6 [ call wrapper         ]
  //  -5 [ result               ]
  //  -4 [ result type          ]
  //  -3 [ method               ]
  //  -2 [ entry point          ]
  //  -1 [ parameters           ]
  //   0 [ saved rbp            ] <--- rbp 栈底
  //   1 [ return address       ]
  //   2 [ parameter size       ]
  //   3 [ thread               ]

address generate_call_stub(address& return_address) {
    
    //__为MacroAssembler, 用于生成机器指令
    address start = __ pc();

    //以下所有地址为rbp的偏移量, 存放在栈中
    //在push rbp; mov rsp rbp;后rbp的偏移量
    const Address rsp_after_call(rbp, rsp_after_call_off * wordSize);

    const Address call_wrapper  (rbp, call_wrapper_off   * wordSize);
    const Address result        (rbp, result_off         * wordSize);
    const Address result_type   (rbp, result_type_off    * wordSize);
    const Address method        (rbp, method_off         * wordSize);
    const Address entry_point   (rbp, entry_point_off    * wordSize);
    const Address parameters    (rbp, parameters_off     * wordSize);
    const Address parameter_size(rbp, parameter_size_off * wordSize);

    const Address thread        (rbp, thread_off         * wordSize);

    const Address r15_save(rbp, r15_off * wordSize);
    const Address r14_save(rbp, r14_off * wordSize);
    const Address r13_save(rbp, r13_off * wordSize);
    const Address r12_save(rbp, r12_off * wordSize);
    const Address rbx_save(rbp, rbx_off * wordSize);

    // 开始生成桩代码
    __ enter();//push rbp; mov rsp rbp;
    __ subptr(rsp, -rsp_after_call_off * wordSize);//栈顶扩展

    //将调用参数入栈
    __ movptr(parameters,   c_rarg5); // parameters
    __ movptr(entry_point,  c_rarg4); // entry_point

    __ movptr(method,       c_rarg3); // method
    __ movl(result_type,  c_rarg2);   // result type
    __ movptr(result,       c_rarg1); // result
    __ movptr(call_wrapper, c_rarg0); // call wrapper

    // 以下几个寄存器遵循被调用者保存
    __ movptr(rbx_save, rbx);
    __ movptr(r12_save, r12);
    __ movptr(r13_save, r13);
    __ movptr(r14_save, r14);
    __ movptr(r15_save, r15);

    //将线程保存在寄存器中
    __ movptr(r15_thread, thread);
    __ reinit_heapbase();

    Label loop;
    __ movptr(c_rarg2, parameters);       // 参数指针保存在
    __ movl(c_rarg1, c_rarg3);            // 参数数量保存在c_rarg1寄存器中
    __ BIND(loop);                      //循环跳转点
    __ movptr(rax, Address(c_rarg2, 0));// 获取参数
    __ addptr(c_rarg2, wordSize);       // 指向下一个参数
    __ decrementl(c_rarg1);             // 减少参数数量用于结束判断
    __ push(rax);                       // 参数压栈
    __ jcc(Assembler::notZero, loop);   //判断是否结束参数压栈

    // call Java function
    __ BIND(parameters_done);           
    __ movptr(rbx, method);             // 将方法指针存入寄存器
    __ movptr(c_rarg1, entry_point);    // 入口点
    __ mov(r13, rsp);                   // 保存栈顶
    __ call(c_rarg1);                   //调用方法

    BLOCK_COMMENT("call_stub_return_address:");
    return_address = __ pc();//待返回的桩代码入口地址

    
    //保存结果, 除了object long float double都已int保存
    __ movptr(c_rarg0, result);
    Label is_long, is_float, is_double, exit;
    __ movl(c_rarg1, result_type);
    __ cmpl(c_rarg1, T_OBJECT);
    __ jcc(Assembler::equal, is_long);
    __ cmpl(c_rarg1, T_LONG);
    __ jcc(Assembler::equal, is_long);
    __ cmpl(c_rarg1, T_FLOAT);
    __ jcc(Assembler::equal, is_float);
    __ cmpl(c_rarg1, T_DOUBLE);
    __ jcc(Assembler::equal, is_double);

    __ movl(Address(c_rarg0, 0), rax);

    __ BIND(exit);

    //弹出参数
    __ lea(rsp, rsp_after_call);

    //还原寄存器
    __ movptr(r15, r15_save);
    __ movptr(r14, r14_save);
    __ movptr(r13, r13_save);
    __ movptr(r12, r12_save);
    __ movptr(rbx, rbx_save);

    //重置栈顶
    __ addptr(rsp, -rsp_after_call_off * wordSize);

    // 返回
    __ vzeroupper();
    __ pop(rbp);
    __ ret(0);

    // 处理非int返回值
    __ BIND(is_long);
    __ movq(Address(c_rarg0, 0), rax);
    __ jmp(exit);

    __ BIND(is_float);
    __ movflt(Address(c_rarg0, 0), xmm0);
    __ jmp(exit);

    __ BIND(is_double);
    __ movdbl(Address(c_rarg0, 0), xmm0);
    __ jmp(exit);

    return start;
}
```

以上为大致流程, 关于MacroAssembler的相关需要x86汇编指示, 暂未看完

## 实验
``` c++
//模拟用mmap申请一块可执行内存, 直接写指令, 用c调用

#include <stdio.h>
//#include <malloc.h>
#include <sys/mman.h>
#include <unistd.h>
#include <malloc.h>

typedef signed char int_8;

int_8* end;
int_8* start;
int_8* curr;
typedef int (*sum)(int&, int&);

void emit(int x) {
  *curr = x;
  curr++;
}

void printCode() {
  int_8* tmp = start;
  while(tmp != end) {
    printf("%p: %X\n", tmp, *tmp);
    ++tmp;
  }
}

int main() {
  size_t size = 200;
  size_t pagesize = getpagesize();
  curr = start = (int_8*)mmap(
    (void*)(pagesize * (1 << 20)), 
    pagesize, 
    PROT_EXEC|PROT_WRITE|PROT_READ, 
     MAP_ANON | MAP_PRIVATE, 0, 0);
    //curr = start = (int_8*)malloc(200);
  end = (int_8*)((size_t)start + size);
  printf("start:%p, end:%p\n", start, end);
   emit(0x55);//push %rbp
   emit(0x48);
  emit(0x89);
   emit(0xe5);//mov %rsp,%rbp
  emit(0x48);
  emit(0x89);
  emit(0x7d);
  emit(0xf8);//mov %rdi,-0x8(%rbp)
  emit(0x48);
  emit(0x89);
  emit(0x75);
  emit(0xf0);//mov %rsi,-0x10(%rbp)
  emit(0x48);
  emit(0x8b);
  emit(0x45);
  emit(0xf8);//mov 0x8(%rbp),%rax
  emit(0x8b);
  emit(0x10);//mov (rax),%edx
  emit(0x48);
  emit(0x8b);
  emit(0x45);
  emit(0xf0);//mov -0x10(%rbp),%rax
  emit(0x8b);
  emit(0x00);//mov (%rax),%eax
  emit(0x01);
  emit(0xd0);//add %edx,%eax
  emit(0x5d);//pop %rbp
  emit(0xc3);//retq
  //curr--;
  int a = 3;
  int b = 5;
  sum s = reinterpret_cast<sum>(start);
  int c = s(a, b);
  printf("c:%d\n", c);//8
  //printCode();
}
``` 