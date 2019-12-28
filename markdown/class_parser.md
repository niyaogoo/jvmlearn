# class文件解析
hotspot/share/classfile/classFileParser.hpp负责解析class文件

## ClassFileParser::parse_stream
顺序解析:
* 4字节魔数: 0xCAFEBABE
* 2字节大版本: 已定义的版本号49(JDK1.5)->58(JDK14)
* 2字节小版本
* 2字节常量个数: 解析到这会在元空间中分配常量池, 最多0xFFFF(ConstantPool)并开始解析常量
  * 解析常量:将class中的常量解析至常量池(ConstantsPool)常量长度由tag(1字节)+n(根据tag类型确定)决定
  * 验证常量:循环常量池(ConstantsPool), 检查是否正确, 并对常量做一些处理
* 在元空间分配class指针空间
* 2字节class的ACC并验证
* 2字节SuperClassIndex
* 2字节interface长度
  * 解析interface
* 解析field
  * field顺序:[access, name index, sig index, initial value index, low_offset, high_offset]
  * 解析attributes
* 解析method
  * //TODO too huge
* 解析attribute
* field,method,class的attribute
### 常量池解析(ConstantPool)
#### 结构
_tags:Array, 储存常量类型

ConstantsBody: 未在结构中申明, 在分配ConstantPool时根据常量数量会预留空间, 内容为8字节类型或指针;
```c++
intptr_t* base() const { return (intptr_t*) (((char*) this) + sizeof(ConstantPool)); }
```

#### 解析class文件中的常量至常量池
```c++
// hotspot/share/oops/constantPool.hpp

//..
//解析类型为Class
void klass_index_at_put(int which, int name_index) {
    //tag标记为JVM_CONSTANT_ClassIndex, 等待下一步验证时进一步处理
    tag_at_put(which, JVM_CONSTANT_ClassIndex);
    //body为name的index, 如java/lang/StringBuilder
    *int_at_addr(which) = name_index;
}

//解析类型NameAndType
void name_and_type_at_put(int which, int name_index, int signature_index) {
    //tag标记为JVM_CONSTANT_NameAndType
    tag_at_put(which, JVM_CONSTANT_NameAndType);
    //index以2字节储存在class文件中, jint为4字节, 左移2字节低2字节与name_index结合储存在常量池body中
    *int_at_addr(which) = ((jint) signature_index<<16) | name_index;
  }
//..
```

#### 验证常量

```c++
//hotspot/share/oops/constantPool.hpp

//当验证tag遇到JVM_CONSTANT_ClassIndex时作以下处理
void unresolved_klass_at_put(int which, int name_index, int resolved_klass_index) {
    //替换tag为JVM_CONSTANT_UnresolvedClass
    release_tag_at_put(which, JVM_CONSTANT_UnresolvedClass);
    //body为class序号(作为高16位)|name索引(作为低16位)
    *int_at_addr(which) =
      build_int_from_shorts((jushort)resolved_klass_index, (jushort)name_index);
}
```

#### 常量类型

```c++
JVM_CONSTANT_Utf8                   = 1,//2字节指定长度n, 后跟n字节数据
JVM_CONSTANT_Unicode                = 2, //未使用
JVM_CONSTANT_Integer                = 3,//4字节int
JVM_CONSTANT_Float                  = 4,//4字节float
JVM_CONSTANT_Long                   = 5,//8字节long
JVM_CONSTANT_Double                 = 6,//8字节double
JVM_CONSTANT_Class                  = 7,//2字节class
JVM_CONSTANT_String                 = 8,//2字节字符串索引
JVM_CONSTANT_Fieldref               = 9,//2字节class索引, 2字节field索引
JVM_CONSTANT_Methodref              = 10,//2字节class索引, 2字节method索引
JVM_CONSTANT_InterfaceMethodref     = 11,//2字节class索引, 2字节method索引
JVM_CONSTANT_NameAndType            = 12,//2字节name索引, 2字节签名索引
JVM_CONSTANT_MethodHandle           = 15,  // JSR 292
JVM_CONSTANT_MethodType             = 16,  // JSR 292
JVM_CONSTANT_Dynamic                = 17,//2字节bsm索引, 2字节name_type索引
JVM_CONSTANT_InvokeDynamic          = 18,//2字节bsm索引, 2字节name_type索引
JVM_CONSTANT_Module                 = 19,//jdk9之后处理, 2字节
JVM_CONSTANT_Package                = 20,//jdk9之后处理, 2字节
JVM_CONSTANT_ExternalMax            = 20 //标记结束
```
举例, javap -v Object
```
Constant pool:
  序号  类型                name index[.type index]
   #1 = Class              #2             // java/lang/StringBuilder
   #2 = Utf8               java/lang/StringBuilder
   #3 = Methodref          #1.#4          // java/lang/StringBuilder."<init>":()V
   #4 = NameAndType        #5:#6          // "<init>":()V
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Methodref          #8.#9          // java/lang/Object.getClass:()Ljava/lang/Class;
   #8 = Class              #10            // java/lang/Object
   #9 = NameAndType        #11:#12        // getClass:()Ljava/lang/Class;
```
* #1类型为Class, 名字索引为#2, 名称为java/lang/StringBuilder
  * #2为字符串:java/lang/StringBuilder
* #3为method引用, class_index为#1,name_and_type_index为#4
  * #1为class_index
  * #4类型为NameAndType, name_index为#5, signature_index为#6
    * #5为字符串\<init\>
    * #6为签名()V
