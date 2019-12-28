# class文件

## class有以下几个状态
* allocated //已分配metaspace
* loaded //已加载至metaspace并且确定继承关系
* linked //已连接已验证
* being_initialized //正在运行初始化
* fully_initialized //最后的成功状态
* initialization_error //初始化错误
instanceKlass被classFileParser加载至metaspace后状态为loaded