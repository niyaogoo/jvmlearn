# Method解析

在解析完field后开始解析method, 首先是两字节的length, 说明方法数量

> hotspot/share/classfile/classFileParser.cpp#parse_method

## Method示例

示例
``` s
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class Hello
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: invokevirtual #4                  // Method hello:()V
        12: return
      LineNumberTable:
        line 3: 0
        line 4: 8
        line 5: 12
```
``` c++
//hotspot/share/classfile/classFileParser.cpp#parse_method
//解析method的主要方法
Method* ClassFileParser::parse_method(const ClassFileStream* const cfs,
                                      bool is_interface,
                                      const ConstantPool* cp,
                                      AccessFlags* const promoted_flags,
                                      TRAPS) {
  //cfs:文件流, is_interface:是否是接口
  //cp:常量符号池, promoted_flagsL: 所属klass的access flags
  assert(cfs != NULL, "invariant");
  assert(cp != NULL, "invariant");
  assert(promoted_flags != NULL, "invariant");

  //线程arena分配标记与回收准备
  ResourceMark rm(THREAD);
  //保证接下来有8字节内容:access_flags, name_index, descriptor_index, attributes_count
  cfs->guarantee_more(8, CHECK_NULL);
  //2字节access_flags
  int flags = cfs->get_u2_fast();
  //2字节方法名称索引
  const u2 name_index = cfs->get_u2_fast();
  //获取常量池长度, 不知道有什么用
  const int cp_size = cp->length();
  //验证方法名称是否存在与符号常量池中, 并且是utf8, msg中的第二个占位符%s在check_property中替换, class file name
  check_property(
    valid_symbol_at(name_index),
    "Illegal constant pool index %u for method name in class file %s",
    name_index, CHECK_NULL);
  //获取方法名称符号
  const Symbol* const name = cp->symbol_at(name_index);
  //验证方法名称,不能为NULL 非<[cl]init>不能<开头
  verify_legal_method_name(name, CHECK_NULL);
  //获取方法签名符号索引并验证
  const u2 signature_index = cfs->get_u2_fast();
  guarantee_property(
    valid_symbol_at(signature_index),
    "Illegal constant pool index %u for method signature in class file %s",
    signature_index, CHECK_NULL);
  //获取方法签名符号
  const Symbol* const signature = cp->symbol_at(signature_index);
  //如果是<clinit>
  if (name == vmSymbols::class_initializer_name()) {
    // We ignore the other access flags for a valid class initializer.
    // (JVM Spec 2nd ed., chapter 4.6)
    //如果jdk7以前忽略其他acc flag
    if (_major_version < 51) { // backward compatibility
      flags = JVM_ACC_STATIC;
    } else if ((flags & JVM_ACC_STATIC) == JVM_ACC_STATIC) {
      //<clinit>只能是static
      flags &= JVM_ACC_STATIC | JVM_ACC_STRICT;
    } else {
      classfile_parse_error("Method <clinit> is not static in class file %s", CHECK_NULL);
    }
  } else {
    //验证方法access_flags
    verify_legal_method_modifiers(flags, is_interface, name, CHECK_NULL);
  }
  //interface不能有<init>方法
  if (name == vmSymbols::object_initializer_name() && is_interface) {
    classfile_parse_error("Interface cannot have a method named <init>, class file %s", CHECK_NULL);
  }

  int args_size = -1;  // only used when _need_verify is true
  if (_need_verify) {
    //verify_legal_method_signature返回参数数量,注意long与double占两个参数
    args_size = ((flags & JVM_ACC_STATIC) ? 0 : 1) +
                 verify_legal_method_signature(name, signature, CHECK_NULL);
    //参数最大个数255
    if (args_size > MAX_ARGS_SIZE) {
      classfile_parse_error("Too many arguments in method signature in class file %s", CHECK_NULL);
    }
  }

  //屏蔽非method使用的access_flags
  AccessFlags access_flags(flags & JVM_RECOGNIZED_METHOD_MODIFIERS);

  // Default values for code and exceptions attribute elements
  //代码与异常属性默认值
  u2 max_stack = 0;
  u2 max_locals = 0;
  u4 code_length = 0;
  const u1* code_start = 0;
  u2 exception_table_length = 0;
  const unsafe_u2* exception_table_start = NULL; // (potentially unaligned) pointer to array of u2 elements
  Array<int>* exception_handlers = Universe::the_empty_int_array();
  u2 checked_exceptions_length = 0;
  const unsafe_u2* checked_exceptions_start = NULL; // (potentially unaligned) pointer to array of u2 elements
  CompressedLineNumberWriteStream* linenumber_table = NULL;
  int linenumber_table_length = 0;
  int total_lvt_length = 0;
  u2 lvt_cnt = 0;
  u2 lvtt_cnt = 0;
  bool lvt_allocated = false;
  u2 max_lvt_cnt = INITIAL_MAX_LVT_NUMBER;
  u2 max_lvtt_cnt = INITIAL_MAX_LVT_NUMBER;
  u2* localvariable_table_length = NULL;
  const unsafe_u2** localvariable_table_start = NULL; // (potentially unaligned) pointer to array of LVT attributes
  u2* localvariable_type_table_length = NULL;
  const unsafe_u2** localvariable_type_table_start = NULL; // (potentially unaligned) pointer to LVTT attributes
  int method_parameters_length = -1;
  const u1* method_parameters_data = NULL;
  bool method_parameters_seen = false;
  bool parsed_code_attribute = false;
  bool parsed_checked_exceptions_attribute = false;
  bool parsed_stackmap_attribute = false;
  // stackmap attribute - JDK1.5
  const u1* stackmap_data = NULL;
  int stackmap_data_length = 0;
  u2 generic_signature_index = 0;
  MethodAnnotationCollector parsed_annotations;
  const u1* runtime_visible_annotations = NULL;
  int runtime_visible_annotations_length = 0;
  const u1* runtime_invisible_annotations = NULL;
  int runtime_invisible_annotations_length = 0;
  const u1* runtime_visible_parameter_annotations = NULL;
  int runtime_visible_parameter_annotations_length = 0;
  const u1* runtime_invisible_parameter_annotations = NULL;
  int runtime_invisible_parameter_annotations_length = 0;
  const u1* runtime_visible_type_annotations = NULL;
  int runtime_visible_type_annotations_length = 0;
  const u1* runtime_invisible_type_annotations = NULL;
  int runtime_invisible_type_annotations_length = 0;
  bool runtime_invisible_annotations_exists = false;
  bool runtime_invisible_type_annotations_exists = false;
  bool runtime_invisible_parameter_annotations_exists = false;
  const u1* annotation_default = NULL;
  int annotation_default_length = 0;

  // Parse code and exceptions attribute
  //解析代码与异常属性
  //属性数量
  u2 method_attributes_count = cfs->get_u2_fast();
  while (method_attributes_count--) {
    cfs->guarantee_more(6, CHECK_NULL);  // method_attribute_name_index, method_attribute_length
    //2字节method属性名称符号索引
    const u2 method_attribute_name_index = cfs->get_u2_fast();
    //4字节method属性长度
    const u4 method_attribute_length = cfs->get_u4_fast();
    check_property(
      valid_symbol_at(method_attribute_name_index),
      "Invalid method attribute name index %u in class file %s",
      method_attribute_name_index, CHECK_NULL);

    //属性名称
    const Symbol* const method_attribute_name = cp->symbol_at(method_attribute_name_index);
    //处理Code
    if (method_attribute_name == vmSymbols::tag_code()) {
      // Parse Code attribute
      if (_need_verify) {
        //本地方法与抽象方法不能有code
        guarantee_property(
            !access_flags.is_native() && !access_flags.is_abstract(),
                        "Code attribute in native or abstract methods in class file %s",
                         CHECK_NULL);
      }
      //一个method只能由一个code属性
      if (parsed_code_attribute) {
        classfile_parse_error("Multiple Code attributes in class file %s",
                              CHECK_NULL);
      }
      parsed_code_attribute = true;
      // Stack size, locals size, and code size
      cfs->guarantee_more(8, CHECK_NULL);
      //2字节栈容量
      max_stack = cfs->get_u2_fast();
      //2字节本地变量
      max_locals = cfs->get_u2_fast();
      //4字节代码长度
      code_length = cfs->get_u4_fast();
      if (_need_verify) {
        //方法签名标记的参数应该小于等于本地变量最大值
        guarantee_property(args_size <= max_locals,
                           "Arguments can't fit into locals in class file %s",
                           CHECK_NULL);
        //代码长度必须在0-65535字节
        guarantee_property(code_length > 0 && code_length <= MAX_CODE_SIZE,
                           "Invalid method Code length %u in class file %s",
                           code_length, CHECK_NULL);
      }
      // Code pointer
      //开始解析code
      code_start = cfs->current();
      assert(code_start != NULL, "null code start");
      //stream确实有code_length的内容
      cfs->guarantee_more(code_length, CHECK_NULL);
      //代码块, 准备处理异常表
      cfs->skip_u1_fast(code_length);

      // Exception handler table
      //方法内部异常处理表
      //2字节异常表长度
      cfs->guarantee_more(2, CHECK_NULL);  // exception_table_length
      exception_table_length = cfs->get_u2_fast();
      if (exception_table_length > 0) {
        //验证异常表start_pc, end_pc, handler_pc, catch_type_index
        //长度为exception_table_length * 8
        exception_table_start = parse_exception_table(cfs,
                                                      code_length,
                                                      exception_table_length,
                                                      CHECK_NULL);
      }
      //省略LineNumberTable(源码映射表), LoadLocalVariableTables(本地变量表), LoadLocalVariableTypeTables(本地变量类型表)

      // Parse additional attributes in code attribute
      // 解析Code属性中的其他属性, LineNumber, LocalVariable等
      cfs->guarantee_more(2, CHECK_NULL);  // code_attributes_count
      //其他属性长度
      u2 code_attributes_count = cfs->get_u2_fast();

      //记录接下来的解析长度, 最后会与method_attribute_length比较
      unsigned int calculated_attribute_length = 0;

      calculated_attribute_length =
          sizeof(max_stack) + sizeof(max_locals) + sizeof(code_length);
      calculated_attribute_length +=
        code_length +
        sizeof(exception_table_length) +
        sizeof(code_attributes_count) +
        exception_table_length *
            ( sizeof(u2) +   // start_pc
              sizeof(u2) +   // end_pc
              sizeof(u2) +   // handler_pc
              sizeof(u2) );  // catch_type_index
      //开始解析Code属性中的其他属性
      while (code_attributes_count--) {
        cfs->guarantee_more(6, CHECK_NULL);  // code_attribute_name_index, code_attribute_length
        const u2 code_attribute_name_index = cfs->get_u2_fast();
        const u4 code_attribute_length = cfs->get_u4_fast();
        calculated_attribute_length += code_attribute_length +
                                       sizeof(code_attribute_name_index) +
                                       sizeof(code_attribute_length);
        check_property(valid_symbol_at(code_attribute_name_index),
                       "Invalid code attribute name index %u in class file %s",
                       code_attribute_name_index,
                       CHECK_NULL);
        //源码行映射
        if (LoadLineNumberTables &&
            cp->symbol_at(code_attribute_name_index) == vmSymbols::tag_line_number_table()) {
          // Parse and compress line number table
          parse_linenumber_table(code_attribute_length,
                                 code_length,
                                 &linenumber_table,
                                 CHECK_NULL);

        } else if (LoadLocalVariableTables &&
                   cp->symbol_at(code_attribute_name_index) == vmSymbols::tag_local_variable_table()) {
          //本地变量加载映射
          // Parse local variable table
          if (!lvt_allocated) {
            localvariable_table_length = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, u2,  INITIAL_MAX_LVT_NUMBER);
            localvariable_table_start = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, const unsafe_u2*, INITIAL_MAX_LVT_NUMBER);
            localvariable_type_table_length = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, u2,  INITIAL_MAX_LVT_NUMBER);
            localvariable_type_table_start = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, const unsafe_u2*, INITIAL_MAX_LVT_NUMBER);
            lvt_allocated = true;
          }
          if (lvt_cnt == max_lvt_cnt) {
            max_lvt_cnt <<= 1;
            localvariable_table_length = REALLOC_RESOURCE_ARRAY(u2, localvariable_table_length, lvt_cnt, max_lvt_cnt);
            localvariable_table_start  = REALLOC_RESOURCE_ARRAY(const unsafe_u2*, localvariable_table_start, lvt_cnt, max_lvt_cnt);
          }
          localvariable_table_start[lvt_cnt] =
            parse_localvariable_table(cfs,
                                      code_length,
                                      max_locals,
                                      code_attribute_length,
                                      &localvariable_table_length[lvt_cnt],
                                      false,    // is not LVTT
                                      CHECK_NULL);
          total_lvt_length += localvariable_table_length[lvt_cnt];
          lvt_cnt++;
        } else if (LoadLocalVariableTypeTables &&
                   _major_version >= JAVA_1_5_VERSION &&
                   cp->symbol_at(code_attribute_name_index) == vmSymbols::tag_local_variable_type_table()) {
          //本地变量类型映射表
          if (!lvt_allocated) {
            localvariable_table_length = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, u2,  INITIAL_MAX_LVT_NUMBER);
            localvariable_table_start = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, const unsafe_u2*, INITIAL_MAX_LVT_NUMBER);
            localvariable_type_table_length = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, u2,  INITIAL_MAX_LVT_NUMBER);
            localvariable_type_table_start = NEW_RESOURCE_ARRAY_IN_THREAD(
              THREAD, const unsafe_u2*, INITIAL_MAX_LVT_NUMBER);
            lvt_allocated = true;
          }
          // Parse local variable type table
          if (lvtt_cnt == max_lvtt_cnt) {
            max_lvtt_cnt <<= 1;
            localvariable_type_table_length = REALLOC_RESOURCE_ARRAY(u2, localvariable_type_table_length, lvtt_cnt, max_lvtt_cnt);
            localvariable_type_table_start  = REALLOC_RESOURCE_ARRAY(const unsafe_u2*, localvariable_type_table_start, lvtt_cnt, max_lvtt_cnt);
          }
          localvariable_type_table_start[lvtt_cnt] =
            parse_localvariable_table(cfs,
                                      code_length,
                                      max_locals,
                                      code_attribute_length,
                                      &localvariable_type_table_length[lvtt_cnt],
                                      true,     // is LVTT
                                      CHECK_NULL);
          lvtt_cnt++;
        } else if (_major_version >= Verifier::STACKMAP_ATTRIBUTE_MAJOR_VERSION &&
                   cp->symbol_at(code_attribute_name_index) == vmSymbols::tag_stack_map_table()) {
          //方法栈映射表
          // Stack map is only needed by the new verifier in JDK1.5.
          if (parsed_stackmap_attribute) {
            classfile_parse_error("Multiple StackMapTable attributes in class file %s", CHECK_NULL);
          }
          stackmap_data = parse_stackmap_table(cfs, code_attribute_length, _need_verify, CHECK_NULL);
          stackmap_data_length = code_attribute_length;
          parsed_stackmap_attribute = true;
        } else {
          //未知属性
          // Skip unknown attributes
          cfs->skip_u1(code_attribute_length, CHECK_NULL);
        }
      }
      // check method attribute length
      //验证已解析长度是否和method_attribute_length是否一样
      if (_need_verify) {
        guarantee_property(method_attribute_length == calculated_attribute_length,
                           "Code segment has wrong length in class file %s",
                           CHECK_NULL);
      }
    } else if (method_attribute_name == vmSymbols::tag_exceptions()) {
      //解析异常属性
      // Parse Exceptions attribute
      if (parsed_checked_exceptions_attribute) {
        classfile_parse_error("Multiple Exceptions attributes in class file %s",
                              CHECK_NULL);
      }
      parsed_checked_exceptions_attribute = true;
      checked_exceptions_start =
            parse_checked_exceptions(cfs,
                                     &checked_exceptions_length,
                                     method_attribute_length,
                                     CHECK_NULL);
    }
    //剩下的先跳过, 比较重要的是annotation那一块, 现在主要看Code


```

### 解析2字节flag
标记访问修饰符, 如public
### 解析2字节name索引
如hello, 验证方法名称
### 解析2字节方法签名索引
如([Ljava/lang/String;)V
### 判断如果


()参数列表, [数组, Lx/x/x;对象, V返回void


## 注意
一个Method符号由flag与name_index组成, flag为访问

* 两个特殊的method
  * <init> 对象初始化方法
  * <clinit> class初始化方法

* ClassFileParser::verify_legal_method_signature

验证方法签名并返回参数数量, 注意J(long)与D(double)占两个参数