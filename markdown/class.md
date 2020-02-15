# class文件

FieldType term	Type	Interpretation
B	byte	signed byte
C	char	Unicode character code point in the Basic Multilingual Plane, encoded with UTF-16
D	double	double-precision floating-point value
F	float	single-precision floating-point value
I	int	integer
J	long	long integer
L ClassName ;	reference	an instance of class ClassName
S	short	signed short
Z	boolean	true or false
[	reference	one array dimension

## class有以下几个状态
* allocated //已分配metaspace
* loaded //已加载至metaspace并且确定继承关系
* linked //已连接已验证
* being_initialized //正在运行初始化
* fully_initialized //最后的成功状态
* initialization_error //初始化错误
instanceKlass被classFileParser加载至metaspace后状态为loaded