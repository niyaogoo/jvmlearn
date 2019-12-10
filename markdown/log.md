# Hotspot日志源码学习

Hotspot中的日志接口在hotspot.share.logging.log.hpp中定义, 日志级别分为Trace, Debug, Info, Warning, Error;
支持以下特性:
* 日志级别由高向低传递(Trace最低, Error最高);
* 日志级别可以绑定多个输出;
* 日志标签组合, 用以区分打印前缀与格式;

hotspot中的日志调用方式如下(*):
```c++
//hotspot->share->memory->universe.cpp

jint Universe::initialize_heap() {
  assert(_collectedHeap == NULL, "Heap already created");
  _collectedHeap = GCConfig::arguments()->create_heap();
  jint status = _collectedHeap->initialize();

  if (status == JNI_OK) {
    log_info(gc)("Using %s", _collectedHeap->name());//*
  }

  return status;
}
```
## 宏扩展
从入口开始阅读, log_info为trace至error五个中的一个, 其中大多数代码为宏与模板特化, 将log.hpp与logTag.hpp中涉及到的代码合并显示, 方便阅读, 大多数为宏, 真正实现为LogImpl:
```c++
//hotspot->share->logging->log.hpp

//
// Logging macros
//
// Usage:
//   log_<level>(<comma separated log tags>)(<printf-style log arguments>);
// e.g.
//   log_debug(logging)("message %d", i);
//
// Note that these macros will not evaluate the arguments unless the logging is enabled.
//
#define log_error(...)   (!log_is_enabled(Error, __VA_ARGS__))   ? (void)0 : LogImpl<LOG_TAGS(__VA_ARGS__)>::write<LogLevel::Error>
#define log_warning(...) (!log_is_enabled(Warning, __VA_ARGS__)) ? (void)0 : LogImpl<LOG_TAGS(__VA_ARGS__)>::write<LogLevel::Warning>
#define log_info(...)    (!log_is_enabled(Info, __VA_ARGS__))    ? (void)0 : LogImpl<LOG_TAGS(__VA_ARGS__)>::write<LogLevel::Info>
#define log_debug(...)   (!log_is_enabled(Debug, __VA_ARGS__))   ? (void)0 : LogImpl<LOG_TAGS(__VA_ARGS__)>::write<LogLevel::Debug>
#define log_trace(...)   (!log_is_enabled(Trace, __VA_ARGS__))   ? (void)0 : LogImpl<LOG_TAGS(__VA_ARGS__)>::write<LogLevel::Trace>
//...
//...
// Convenience macro to test if the logging is enabled on the specified level for given tags.
#define log_is_enabled(level, ...) (LogImpl<LOG_TAGS(__VA_ARGS__)>::is_level(LogLevel::level))
```

```c++
//hotspot->share->logging->logTag.hpp

// List of available logging tags. New tags should be added here.
// (The tags 'all', 'disable' and 'help' are special tags that can
// not be used in log calls, and should not be listed below.)
#define LOG_TAG_LIST \
  LOG_TAG(add) \
  LOG_TAG(age) \
  LOG_TAG(alloc) \
  LOG_TAG(aot) \
  LOG_TAG(annotation) \
  LOG_TAG(arguments) \
  LOG_TAG(attach) \
  LOG_TAG(barrier) \
  LOG_TAG(biasedlocking) \
  LOG_TAG(blocks) \
  LOG_TAG(bot) \
  //...

#define PREFIX_LOG_TAG(T) (LogTag::_##T)

// Expand a set of log tags to their prefixed names.
// For error detection purposes, the macro passes one more tag than what is supported.
// If too many tags are given, a static assert in the log class will fail.
#define LOG_TAGS_EXPANDED(T0, T1, T2, T3, T4, T5, ...)  PREFIX_LOG_TAG(T0), PREFIX_LOG_TAG(T1), PREFIX_LOG_TAG(T2), \
                                                        PREFIX_LOG_TAG(T3), PREFIX_LOG_TAG(T4), PREFIX_LOG_TAG(T5)
// The EXPAND_VARARGS macro is required for MSVC, or it will resolve the LOG_TAGS_EXPANDED macro incorrectly.
#define EXPAND_VARARGS(x) x
#define LOG_TAGS(...) EXPAND_VARARGS(LOG_TAGS_EXPANDED(__VA_ARGS__, _NO_TAG, _NO_TAG, _NO_TAG, _NO_TAG, _NO_TAG, _NO_TAG))

// Log tags are used to classify log messages.
// Each log message can be assigned between 1 to LogTag::MaxTags number of tags.
// Specifying multiple tags for a log message means that only outputs configured
// for those exact tags, or a subset of the tags with a wildcard, will see the logging.
// Multiple tags should be comma separated, e.g. log_error(tag1, tag2)("msg").
class LogTag : public AllStatic {
 public:
  // The maximum number of tags that a single log message can have.
  // E.g. there might be hundreds of different tags available,
  // but a specific log message can only be tagged with up to MaxTags of those.
  static const size_t MaxTags = 5;

  enum type {
    __NO_TAG,
#define LOG_TAG(name) _##name,
    LOG_TAG_LIST
#undef LOG_TAG
    Count
  };
```
class LogTag中的type定义了所有的可用标签, 宏扩展后为__add, __age, __gc等.

log_info与log_is_enabled宏中的LOG_TAGS的作用就是特化所有标签的组合, 确定具体的LogImpl, 标签最多支持五种组合, LOG_TAGS_EXPANDED定了了六个标签, 最有一个是用来检查参数是否正确, 必须为__NO_TAG.

## LogImpl代码
接下来具体查看LogImpl代码, 首先是is_level, 该方法为static, 用来确定是否支持当前日志级别:
```c++
template <LogTagType T0, LogTagType T1 = LogTag::__NO_TAG, LogTagType T2 = LogTag::__NO_TAG, LogTagType T3 = LogTag::__NO_TAG,
          LogTagType T4 = LogTag::__NO_TAG, LogTagType GuardTag = LogTag::__NO_TAG>
class LogImpl {
 private:
  static const size_t LogBufferSize = 512;
 public:
  // Make sure no more than the maximum number of tags have been given.
  // The GuardTag allows this to be detected if/when it happens. If the GuardTag
  // is not __NO_TAG, the number of tags given exceeds the maximum allowed.
  STATIC_ASSERT(GuardTag == LogTag::__NO_TAG); // Number of logging tags exceeds maximum supported!

  // Empty constructor to avoid warnings on MSVC about unused variables
  // when the log instance is only used for static functions.
  LogImpl() {
  }

  static bool is_level(LogLevelType level) {
    return LogTagSetMapping<T0, T1, T2, T3, T4>::tagset().is_level(level);
  }
```
可以看到其实是调用LogTagSetMapping中的tagset().is_level(level), 查看LogTagSetMapping如何实现:
```c++
template <LogTagType T0, LogTagType T1 = LogTag::__NO_TAG, LogTagType T2 = LogTag::__NO_TAG,
          LogTagType T3 = LogTag::__NO_TAG, LogTagType T4 = LogTag::__NO_TAG,
          LogTagType GuardTag = LogTag::__NO_TAG>
class LogTagSetMapping : public AllStatic {
private:
  // Verify number of logging tags does not exceed maximum supported.
  STATIC_ASSERT(GuardTag == LogTag::__NO_TAG);
  static LogTagSet _tagset;

public:
  static LogTagSet& tagset() {
    return _tagset;
  }
};

// Instantiate the static field _tagset for all tagsets that are used for logging somewhere.
// (This must be done here rather than the .cpp file because it's a template.)
// Each combination of tags used as template arguments to the Log class somewhere (via macro or not)
// will instantiate the LogTagSetMapping template, which in turn creates the static field for that
// tagset. This _tagset contains the configuration for those tags.
template <LogTagType T0, LogTagType T1, LogTagType T2, LogTagType T3, LogTagType T4, LogTagType GuardTag>
LogTagSet LogTagSetMapping<T0, T1, T2, T3, T4, GuardTag>::_tagset(&LogPrefix<T0, T1, T2, T3, T4>::prefix, T0, T1, T2, T3, T4);

```
LogTagSetMapping也是一个模板类, 作用也很简单, 根据不同的标签组合初始化不同的static LogTagSet _tagset, 看一下LogTagSet的实现
```c++
// The tagset represents a combination of tags that occur in a log call somewhere.
// Tagsets are created automatically by the LogTagSetMappings and should never be
// instantiated directly somewhere else.
class LogTagSet {
 private:
  static LogTagSet* _list;
  static size_t _ntagsets;

  LogTagSet* const _next;
  size_t _ntags;
  LogTagType _tag[LogTag::MaxTags];

  LogOutputList _output_list;
  LogDecorators _decorators;

  typedef size_t (*PrefixWriter)(char* buf, size_t size);
  PrefixWriter _write_prefix;

  // Keep constructor private to prevent incorrect instantiations of this class.
  // Only LogTagSetMappings can create/contain instances of this class.
  // The constructor links all tagsets together in a global list of tagsets.
  // This list is used during configuration to be able to update all tagsets
  // and their configurations to reflect the new global log configuration.
  LogTagSet(PrefixWriter prefix_writer, LogTagType t0, LogTagType t1, LogTagType t2, LogTagType t3, LogTagType t4);
```
* static LogTagSet* _list: 以单向链表存放所有的LogTagSet;
* static size_t _ntagsets: LogTagSet的实例数;
* LogTagSet* const _next: 指向下一个LogTagSet实例;
* size_t _ntags: 当前的标签数;
* LogTagType _tag[]: 当前标签;
* LogOutputList _output_list: 日志输出管理主要实现, 下面细说;
* LogDecorators _decorators: 日志前缀与格式装饰器;

LogTagSet的默认构造函数
```c++
// This constructor is called only during static initialization.
// See the declaration in logTagSet.hpp for more information.
LogTagSet::LogTagSet(PrefixWriter prefix_writer, LogTagType t0, LogTagType t1, LogTagType t2, LogTagType t3, LogTagType t4)
    : _next(_list), _write_prefix(prefix_writer) {
  _tag[0] = t0;
  _tag[1] = t1;
  _tag[2] = t2;
  _tag[3] = t3;
  _tag[4] = t4;
  for (_ntags = 0; _ntags < LogTag::MaxTags && _tag[_ntags] != LogTag::__NO_TAG; _ntags++) {
  }
  _list = this;
  _ntagsets++;

  // Set the default output to warning and error level for all new tagsets.
  _output_list.set_output_level(&StdoutLog, LogLevel::Default);
}
```
比较简单, 主要是_output_list.set_output_level, LogLevel::Default值与LogLevel::Warning一样, 所以默认输出级别为Warning, 输出对象为StdoutLog, 看一下_output_list.setoutput_level实现
```c++
//hotspot->share->logging->logOutputList.hpp

// Data structure to keep track of log outputs for a given tagset.
// Essentially a sorted linked list going from error level outputs
// to outputs of finer levels. Keeps an index from each level to
// the first node in the list for the corresponding level.
// This allows a log message on, for example, info level to jump
// straight into the list where the first info level output can
// be found. The log message will then be printed on that output,
// as well as all outputs in nodes that follow in the list (which
// can be additional info level outputs and/or debug and trace outputs).
//
// Each instance keeps track of the number of current readers of the list.
// To remove a node from the list the node must first be unlinked,
// and the memory for that node can be freed whenever the removing
// thread observes an active reader count of 0 (after unlinking it).
class LogOutputList {
 private:
  struct LogOutputNode : public CHeapObj<mtLogging> {
    LogOutput*      _value;
    LogOutputNode*  _next;
    LogLevelType    _level;
  };

  LogOutputNode*  _level_start[LogLevel::Count];
  //..
  //..

//hotspot->share->logging->logOutputList.cpp

void LogOutputList::set_output_level(LogOutput* output, LogLevelType level) {
  LogOutputNode* node = find(output);
  if (level == LogLevel::Off && node != NULL) {
    remove_output(node);
  } else if (level != LogLevel::Off && node == NULL) {
    add_output(output, level);
  } else if (node != NULL) {
    update_output_level(node, level);
  }
}

void LogOutputList::add_output(LogOutput* output, LogLevelType level) {
  LogOutputNode* node = new LogOutputNode();
  node->_value = output;
  node->_level = level;

  // Set the next pointer to the first node of a lower level
  for (node->_next = _level_start[level];
       node->_next != NULL && node->_next->_level == level;
       node->_next = node->_next->_next) {
  }

  // Update the _level_start index
  for (int l = LogLevel::Last; l >= level; l--) {
    if (_level_start[l] == NULL || _level_start[l]->_level < level) {
      _level_start[l] = node;
    }
  }

  // Add the node the list
  for (LogOutputNode* cur = _level_start[LogLevel::Last]; cur != NULL; cur = cur->_next) {
    if (cur != node && cur->_next == node->_next) {
      cur->_next = node;
      break;
    }
  }
}
```
结合注释总结LogOutputList:
* 为每一个日志级别维护了一个入口, _level_start[];
* 每一个日志级别包含下一级的日志Node;
* 同一个日志级别可以绑定多个输出;

对LogOutputNode的添加、删除、更新的代码主要都在add_output()中, 3个for循环需要结合起来一起看, 
* 第一个for将当前添加的日志级别的_next指针指向下一个日志级别.(第一个for只完成了一半, 另外一半在第二个for中完成)
* 第二个for将当前添加的日志级别至Error的所有_level_start更新(前提是_level_start中的日志级别小于当前日志级别, 也就是说_level_start入口永远是当前设置的最高级别).
* 第三个for将当前添加的日志设置在一个合适的位置(由于Error为最高级别, 一定包含所有其他日志级别,所以只需要设置_level_start\[LogLevel::Error\]即可, LogLevel::Last值与LogLevel::Error相同)

看一下图例
> 默认日志级别Warning, 定义一个输出节点WarningNode, 初始化后_level_start如下:

| _level_start | \[0\]Trace] | \[1\]Debug | \[2\]Info | \[3\]Warning | \[4\]Error |
|-|-|-|-|-|-|-|
||NULL|NULL|NULL|WarningNode|WarningNode|
|_next|NULL|NULL|NULL|NULL|NULL|

>现在需要设置一个日志级别为Info, 定义一个输出节点InfoNode, 调用add_output后如下:

| _level_start | \[0\]Trace] | \[1\]Debug | \[2\]Info | \[3\]Warning | \[4\]Error |
|-|-|-|-|-|-|-|
||NULL|NULL|InfoNode|WarningNode|WarningNode|
|_next|NULL|NULL|NULL|InfoNode|InfoNode|

>再定义一个ErrorNode, 调用add_output后如下:

| _level_start | \[0\]Trace] | \[1\]Debug | \[2\]Info | \[3\]Warning | \[4\]Error |
|-|-|-|-|-|-|-|
||NULL|NULL|InfoNode|WarningNode|ErrorNode|
|_next|NULL|NULL|NULL|InfoNode|WarningNode|
|_next|NULL|NULL|NULL|NULL|InfoNode|

>最后需要Info有两个输出, 定义一个InfoNode@2, 调用add_output后如下:

| _level_start | \[0\]Trace] | \[1\]Debug | \[2\]Info | \[3\]Warning | \[4\]Error |
|-|-|-|-|-|-|-|
||NULL|NULL|InfoNode|WarningNode|ErrorNode|
|_next|NULL|NULL|InfoNode@2|InfoNode|WarningNode|
|_next|NULL|NULL|NULL|InfoNode@2|InfoNode|
|_next|NULL|NULL|NULL|NULL|InfoNode@2|

这样做的好处可以直接跳转至日志级别的入口, 并且日志级别具有传递性, 同时同个级别可以有多个输出.

剩下就比较简单了, 回到开始:
```c++
#define log_info(...)    (!log_is_enabled(Info, __VA_ARGS__))    ? (void)0 : LogImpl<LOG_TAGS(__VA_ARGS__)>::write<LogLevel::Info>
// Convenience macro to test if the logging is enabled on the specified level for given tags.
#define log_is_enabled(level, ...) (LogImpl<LOG_TAGS(__VA_ARGS__)>::is_level(LogLevel::level))
```
```c++
 // Test if the outputlist has an output for the given level.
  bool is_level(LogLevelType level) const {
    return _level_start[level] != NULL;
  }
```
LogImpl::is_level只需要判断_level_start对应的日志级别是否为NULL就可以判断当前日志级别是否开启.

剩下的就是打印日志, 比较简单, 从_level_start中得到日志Node链表, 遍历并调用write即可.

