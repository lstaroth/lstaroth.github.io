#### glog日志宏定义设计分析与改进探讨

> 看到提交的PR被glog合并到0.7版本后就想着整理下之前写了一半的glog宏定义设计草稿，当初因为glog日志在特定环境下打印失败就去分析了下他的实现逻辑，结果看到宏定义绕来绕去的逻辑就懵了，好不容易分析完代码逻辑，草稿写了一半，发现根本没必要写这么复杂，就在发文章前提了个PR...阴差阳错成了个glog的小contributor

分析下为啥这么设计，想要解决的是什么问题，以及提交pr后发现bug的过程...

#### 发布版本的日志信息隐藏需求
需求不必多说，两个点

1. 避免发布版本因为大量日志字符串导致大小膨胀
2. 避免发布版本大量明文字符串导致代码更容易被分析定位

#### 非流式日志隐藏数据的实现
根据之前的使用的经验来看大多数C风格的日志库都可以保证对外发布版本中的日志信息隐藏，流式日志库就有相当多不支持此特性。想必其中的实现难度是有所区别的。先看下非流式日志如何实现发布版本的日志信息隐藏，这里确实很简单，如下代码所示
```cpp
#ifdef LOG_ON
#define log(format,...) printf(format,__VA_ARGS__)
#else
#define log(format,...)
#endif
```
这种非流式日志的形式，一行日志所需的所有信息均可以被宏参数捕捉并在关闭日志的情况下全部丢弃，而宏的替换发生于编译之前，在编译的时候已经丢失了相关数据，从而保证二进制中不包含此类字符串
#### 流式日志能用宏实现上述机制吗
```cpp
#define log() CreateLogObj()

log() << "test:" << 123;
```
如上代码所示，流式日志宏定义不可能包含所有参数，或者说流式日志只负责返回一个可以接受operator<<的对象，其没法掌控所有的日志参数，自然没法像非流式日志那样在宏定义的阶段完成全部信息的丢弃
估计就是这样的原因导致有些流式日志库没有支持这个特性，但**宏定义阶段不能完成日志信息丢弃可以尝试利用编译期优化的特性丢掉日志**
#### 如何利用编译期优化实现目的
观察需求，```log() << "test:" << 123;```这里需要调整log()这个宏定义的输出，让其整体作为一个表达式被优化掉，即进入一个永假分支
#### 三目表达式构造永假分支
```cpp
true ? (void)0 : (void)(std::cout << "test1: " << 123 << 456);
false ? (void)0 : (void)(std::cout << "test2: " << 123 << 456);
```
测试关闭优化下："test1" & "test2"字符串均出现在二进制文件中
开启优化条件下：只有"test2"字符串出现在二进制文件中
测试结果符合预期，这里采用三目表达式构造永假分支可以解决上述流式输入的信息隐藏问题，接下来就是如何用LOG宏实现上述语法
#### 这样可以让两者等价吗
```cpp
true ? (void)0 : (void)(std::cout << "test1: " << 123 << 456);

#log() true ? (void)0 : (void)CreateLogObj()
log() << "test1" << 123 << 456;
```
显然是不行的，手动展开上述第4行代码为```true ? (void)0 : (void)CreateLogObj() << "test1" << 123 << 456;```，这里会发现两个问题
1. void强制转换了CreateLogObj()的返回值
2. void强制转换没有对整个流生效

如果展开后的形式为```true ? (void)0 : (void)(CreateLogObj() << "test1" << 123 << 456);```加个括号改变下优先级即可同时解决上述两个问题，但显然宏定义是没办法实现在这里加入括号的，glog的解决方法是使用一个运算符代替手动添加的括号改变优先级
#### 简化理解glog宏的全机制
```cpp
struct LogMessageVoidify{
	void operator&(std::ostream&) {}
};

#define LOG() std::cout
#define DLOG() true ? (void) 0 : LogMessageVoidify() & LOG()

true ? (void)0 : (void)(std::cout << "test1: " << 123 << 456);
DLOG() << "test2: " << 123 << 456;
//上式展开后为
true ? (void) 0 : LogMessageVoidify() & LOG() << "test2: " << 123 << 456;
```
operator&的运算符(注意这里是双目的&运算符)优先级低于 <<且高于?:,因此Log()创建的日志流对象先循环迭代将所有日志参数全部执行完成，再作为LogMessageVoidify::operator&的参数输入，并返回void类型的变量,从而满足语法条件通过编译，并在编译过程中作为永假分支被优化掉完成日志信息丢弃
####  借助glog的逻辑快速实现安全的非流式日志的流式输入能力
这里会有新的问题，比如
1. 非流式日志默认设计思路是每次调用打印一行日志，因为每次调用包含一行日志完整的信息，每次调用即刻打印
2. 在其中包含一个流式日志类，较为复杂，而且还需要额外处理多线程问题

那Glog是如何感知到行结束，并且无需手动输入换行符的呢：跟进分析源码发现是在LogMessage临时对象的析构函数中完成的。参考Glog的实现方式，引入外部的日志行类，利用其临时变量立刻析构的特性解决流式输入的"行"感知，并在对象析构函数中调用原始日志打印函数避免引入额外的锁

```cpp
class LogStream
{
public:
    LogStream(const char* file, int line);
    template<typename T> LogStream&& operator<<(const T& v);
    ~LogStreamA(){/*xxx*/};
protected:
    int lineNum;
    const char* fileName;
    std::stringstream lineStream;
};
```

#### 可以简化吗？

其实上面已经说了，核心逻辑为需要让流式输入语法不会出现问题，其次就是要让日志在永假分支从而触发编译器优化。抛弃三目表达式使用`if((false))	log()`即可解决问题

`if((false))	log() << a << b << c  << d;` abcd会直接被永假分支优化掉，并且无论这行代码被写在什么位置都不会产生其他的负面影响,试想下代码写到一行中的情况，即

`if((false))	log() << a << b << c  << d; std::cout << "test" << std::endl;`

编译器在这里实际上是隐式的转换成下面这种方式编译的，**因此不会对其他代码产生负面影响**

`if((false))	{log() << a << b << c  << d;} std::cout << "test" << std::endl;`

#### PR合并后发现bug
其实上面以及能看出来了，默认发生下面的转换，整体作为一个分支出去了
`if((false)) log() << a << b << c << d; std::cout << "test" << std::endl;`
`if((false)) {log() << a << b << c << d;} std::cout << "test" << std::endl;`
但少考虑了一种情况就是这个if本身就在上一层if的语句块中，然后日志的if把原本的逻辑else匹配走了，破坏了原有的逻辑，case如下
```cpp
#define NDEBUG
#include "glog/logging.h"

int main()
{
  if (true)
    DLOG(ERROR) log() << "Hello";
  else
    LOG(ERROR) log() << "Bye";
}
```
这样的模式下展开宏
```cpp
int main()
{
  if (true)
    if(false) log() << "Hello";
  else
    if(false) log() << "Bye";
}
```
本意是希望if(false) << "Hello";作为一个整体被优化，但是抢走了顶层if的else，导致破坏了原始逻辑
```cpp
int main()
{
  if (true)
    if(false) log() << "Hello";
  	else
    	if(false) log() << "Bye";
}
```
其实在看到这样的bug的时候也在思考可不可以修改实现打个补丁，但是三目表达式上面分析过了，if又不能用，如果想要实现这样的补丁那得用其他东西借来模拟if(false),其实很容易想得到，比如
```cpp
  if (true)
    while(false) log() << "Hello";
  else
    while(false) log() << "Bye";
```
但是glog还需要这样的控制宏 LOG_IF(severity, condition) if(condition) LOG(severity)，即他真的可能会执行到LOG(severity)，如果采用while的形式则还要引入循环控制，会非常麻烦，可以做但是感觉并没能解决简化实现的逻辑
