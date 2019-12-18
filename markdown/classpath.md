# classpath

## patch-module
依赖--patch-module启动参数,
在ClassLoader::setup_patch_mod_entries中读取patch-module并设置ModuleClassPathList

### ClasspathStream
> hotspot/share/utilites/classpathStream.hpp/cpp
```c++
class ClasspathStream : public StackObj {
  const char* _class_path;//参数中的字符数组
  int _len;//长度
  int _start;//当前长度
  int _end;//当前结束

public:
  ClasspathStream(const char* class_path) {
    _class_path = class_path;
    _len = (int)strlen(class_path);
    _start = 0;
    _end = 0;
  }

  bool has_next() {
    return _start < _len;
  }

  const char* ClasspathStream::get_next() {
    while (_class_path[_end] != '\0' && _class_path[_end] != os::path_separator()[0]) {//循环到结尾或分隔符,linux中为':'
      _end++;
    }
    int path_len = _end - _start;//计算字符长度
    char* path = NEW_RESOURCE_ARRAY(char, path_len + 1);//在ResourceArea中分配数组, 多1位存放\0
    strncpy(path, &_class_path[_start], path_len);//拷贝字符至path
    path[path_len] = '\0';//字符\0结尾

    //至下一个分隔符
    while (_class_path[_end] == os::path_separator()[0]) {
      _end++;
    }
    _start = _end;//将_start, _end都指向新的字符
    return path;
  }
};
```