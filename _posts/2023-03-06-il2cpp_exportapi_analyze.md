### 主题背景
il2cppdumper工具基于静态分析的思路，干扰方式也是静态的，一旦对抗深入到动态分析的地步，最脆弱的点就变成了export api，即il2cpp_xxx这类gameassembly中导出的函数，基于对这些函数的调用与定位即可轻松接近关键逻辑代码/关键解密代码，本文基于实现视角探索il2cpp_xxx函数导出的原因，不导出的后果以及可能的解决思路

### 确认导出函数的关键代码位置
```cpp
    DO_API(r, n, p)             IL2CPP_EXPORT r n p;
    # define IL2CPP_EXPORT __declspec(dllexport)

    DO_API(const Il2CppImage*, il2cpp_get_corlib, ());
```
review libil2cpp代码后确认定义了il2cpp_xxx函数导出的关键宏为DO_API这一套

### 初步思考
1. 这里的关键不是怎么实现的dll导出，而是为什么要导出，如果不导出会出现什么问题？反射逻辑不支持？还是动态方案不支持？
2. 搜索动态查找某个关键函数的代码，看下使用场景到底是什么 il2cpp_domain_get

### 实验去除函数导出反向推导导出必要性
1. 通过对DO_API=>IL2CPP_EXPORT的修改，此处注意不能只修改DO_API，因为部分函数会重复申明，在其他地方会显示声明为IL2CPP_EXPORT的函数形式，所以需要在底层对IL2CPP_EXPORT的定义进行干预
2. 成功在源码编译通过的情况下去除了所有il2cpp函数的导出，但是运行loader时出错，跟踪报错源头确认unityplayer需要gameassembly的相关函数

### unityplayer使用il2cpp接口原因 & 方式分析
**unityplayer为闭源代码，但官方提供了个pdb，结合ida分析**
1. 字符串搜索确认runtime的加载路径UnityMainImpl()=>LoadScriptingRuntime()=>LoadIl2Cpp()
2. 通过报错信息确认确实为LoadIl2Cpp返回False导致的报错
3. LoadIl2Cpp中会通过LookupSymbol获取导出函数地址
4. 获取的函数统计共有235个，刚好与il2cpp-api-function.h中的DO_API数量吻合，因此确认unityplayer需要确保所有相关函数的导出
5. LookupSymbol没获取到会怎么办？结合ida分析可知不允许出现该情况，一个没获取到就报错
6. 是否所有获取的函数都在unityplayer中有用到?发现大部分的api有被unityplayer包装了一层，但是确实是存在没有被包装的api，比如il2cpp_string_new，但这里可能版本强相关

结论1：如果不修改unityplayer没法做到gameassembly的关键api不导出
结论2：unityplayer闭源，要么购买相关源码自行编译要么采用二进制patch的方式进行修改
结论3：修改的关键点在于如何在确保unityplayer获取函数地址成功的情况下拒绝恶意请求
结论4：另一个关键点在于不能将此问题单独局限于gameassembly中去看，由于unityplayer会有一层wraper，因此后者相当于也是个入口，并且如果在3中采用来源审计去做，则可以通过spoof_call等方式伪造对抗
结论5：有源码可以大改，没源码需要分析关键路径上的关键api，并做好此类api的casebycase的处理

### 其他的可能的il2cpp export api调用者
目前看下来除了unityplayer最可能会调用此类api的就只有动态方案的可能了，重点关注HybridCLR项目，粗略看了下该项目代码，看上去是实现了多matadata和多assembly的兼容性支持，并且相关修改也都在libil2cpp中完成，相关api编译链接即可找到，暂时未发现刚需export的情况

### 看看神奇的Genshin impact是如何做得呢
unityplayer是自己的签名，结合米哈游他们有闭源的unity买断代码，应该是自己魔改了一份给原神用的
看他们的UserAssembly.dll确认相关导出函数只剩下了一个il2cpp_get_api_table，一眼没见过的api，搜了下il2cpp-api-functions确认这个api使他们自己搞得，从名字上大概猜到他们的保护思路了，就是修改unityplayer的LookupSymbol方法，然后去统一获取il2cpp_get_api_table函数，然后再通过该函数鉴权/拿到真正的地址，ida看了下相关函数被VMP了

### 意外发现
github搜索关键字il2cpp_get_api_table发现了个小仓库里面有一些手游的il2cpp的保护分析，值得一看[JMBQ/dump-games](https://github.com/JMBQ/dump-games)，总结如下
月圆之夜:替换dlsym，字符串compare返回地址
龙之谷2:替换dlsym，根据立即数返回地址，一眼就是编译期字符串strhash(参考项目：ADVobfuscator)，隐去了字符串特征，结果特么这天才在报错日志里面暴露了字符串给人定位......

