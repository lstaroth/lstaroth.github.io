### il2cpp的源码修改逻辑

目标平台为win：il2cpp被以源码形式拷贝到导出到工程sln下，修改即可，如果希望所有unity生效的话需要将修改后的文件覆盖回unityeditor中的对应位置

目标平台为IOS：静态链接库libil2cpp.a的形式存在，因此需要手动构建IOS的il2cpp构建工程将其输出文件覆盖替换

### il2cpp模式下的关键文件

1. GameAssembly.dll
2. global-metadata.dat

如果想要知道GameAssembly.dll中定义的关键脚本的string具体值那么得用ida脚本根据index读取global-metadata.dat中的内容才能确定，具体确认逻辑，这是如何实现global-metadata.dat局部加密的关键逻辑点，也是il2cppdumper的关键逻辑

### 定位global-metadata.dat的路径定义点

全局搜索global-metadata.dat，看到底是哪里加载的这个文件，发现在GlobalMetadata.cpp的il2cpp::vm::GlobalMetadata::Initialize中发现关键字，修改后运行游戏失败，此处是修改加载路径的地方，相关代码：

```bash
s_GlobalMetadata = vm::MetadataLoader::LoadMetadataFile("global-metadata.dat");
```

### 分析文件映射逻辑

分析LoadMetadataFile函数，发现其拼接路径，最终拼出完整的global-metadata.dat的加载路径，并以可读的方式打开文件并映射成一份不可写的map地址，如果想要修改为可写需要同时更改open的权限参数以及map的权限参数，在这里我手动给他整个逻辑全改了，换成fopen和fread的方式，对开头的0x100字节进行解密，只要最终返回头指针即可，实测可行(此处不用原始逻辑改map后的内存的原因是这样修改内存会影响到本地文件)

### 分析字面量加载逻辑(官方FPS Demo)

击杀敌人后出现关键字符串：One Enemy left，在global-metadata.dat中搜索此关键字找到位置，将One三个字符+1的形式进行加密，保存后不退进程再进行一次游戏发现提示改为Pof Enemy left，说明这里的读取是运行时的，是实时的读取，那么就有机会被我找到函数中断下来，但还有一点没确认，这里的读取是一次性的统一预读取还是用到的时候再进行读取呢，前者的话可能不太好处理，后者的话就能够实现il2cpp模式下的防global-metadata.dat解密后的dump能力了。

读源码确认这里的关键函数是GetStringLiteralFromIndex，对其下断点，发现返回的newString中出现字符串，堆栈回溯发现确实是il2cpp对C#脚本处理过后的CPP文件直接调用过来的，说明是运行时调用，也就是说可以做到即使部分解密了，尝试了一下确实ok

### 数值定义逻辑：怎么去global-metadata.dat中获取数据

上一步中逐层回溯堆栈，确认逻辑如下(此处以string类型为例分析)

1.il2cpp转换C#为CPP的时候知道哪里是第一次对该变量的引用，插入初始化调用链：

```bash
--il2cpp_codegen_initialize_runtime_metadata
----il2cpp::vm::MetadataCache::InitializeRuntimeMetadata
------il2cpp::vm::GlobalMetadata::InitializeRuntimeMetadata
--------GetStringLiteralFromIndex
```

以上即为string初始化的调用链，继续分析我们发现每个string类型都有一个token，这个token与global-metadata.dat关联，记录着如何找到具体数据，详细过程在上述调用链第三个函数中。

比如这里的值 == 2684354641 == 10100000000000000000000001010001

bit29~bit31为type值：这里为5即kIl2CppMetadataUsageStringLiteral

bit0位initialized位，1表示未初始化，0表示已初始化

bit1~bit28位index位，表示index序号后续用到

找到值的方式，global-metadata.dat头当做GlobalMetadataHeader解析，找到stringLiteralOffset字段，依此为偏移得到Il2CppStringLiteral的数组开始地址，访问index个元素即记录着当前想访问元素的offset和length，已同样的方式找到Header中的stringLiteralDataOffset，加上这样的偏移即为char*的字符串数据了，加上刚刚查到的结构体中的offset即可完成查找，后续可根据此逻辑写完ida解析脚本

### 函数逻辑定义：可以根据定义的id获取函数名吗

上一节中我们根据字符串的定义id找到了对应的字符数据，这里我们跟进一步能不能通过函数的定义id找到函数名呢，这样一来我们就可以使用ida脚本对所有的函数进行命名了，答案是可以

### 关键路径观察：函数名在哪?

上一步中有个typeid控制是否分发给GetStringLiteralFromIndex函数初始化字符内容，同理在InitializeRuntimeMetadata函数内找到了这样的函数调用：GetMethodInfoFromEncodedIndex

此函数内部已相同的规则对token做了处理，即最高3位为type位，最低一位为初始化状态位，其余为index位，然后将分离出的index位传入函数GetMethodInfoFromMethodDefinitionIndex中获取相关函数名信息。最终发现这里的函数名由三部分组成:namespace + name + funcname，如：System.String.copy，前两者在GetTypeInfoFromTypeDefinitionIndex函数返回的typeInfo字段中即可读取到。后者在il2cpp::vm::Class::SetupMethods中查找得到。后面将一一解析

### namespace和name的解析

首先对传入的index调用GetMethodDefinitionFromIndex函数，代码实现为

```cpp
MetadataOffset<const Il2CppMethodDefinition*>(s_GlobalMetadata, s_GlobalMetadataHeader->methodsOffset, index)
```

即找到header中的methodsOffset字段找到Il2CppMethodDefinition数组起始地址，然后找到该函数index的Il2CppMethodDefinition定义，然后访问返回的Il2CppMethodDefinition结构体变量的declaringType字段，并以此调用FromTypeDefinition函数

此函数以如下方式取得此函数申明对应的Il2CppTypeDefinition结构变量typeDefinition

```cpp
MetadataOffset<const Il2CppTypeDefinition*>(s_GlobalMetadata, s_GlobalMetadataHeader->methodsOffset, declaringType)
```

并分别以如下方式获取到name和namespace

```cpp
MetadataOffset<const char*>(s_GlobalMetadata, s_GlobalMetadataHeader->stringOffset, typeDefinition->nameIndex)
MetadataOffset<const char*>(s_GlobalMetadata, s_GlobalMetadataHeader->stringOffset, typeDefinition->namespaceIndex)
```

最后剩下的funcname获取难度稍微复杂些，但大体流程与之前的是一样的

也是typeDefinition结构体中保存有method_count和这个类的第一个method在method中的起始index：methodStart，循环(0,method_count)算出typeDefinition->methodStart + index，即可知道这个typeDefinition定义的所有函数的实际index，并以此调用GetMethodDefinitionFromIndex获取Il2CppMethodDefinition类型的methodDefinition变量，获取方式如下

```cpp
MetadataOffset<const Il2CppMethodDefinition*>(s_GlobalMetadata, s_GlobalMetadataHeader->methodsOffset, index)
```

methodDefinition变量字段nameIndex为StringIndex类型，采用同样的方式获取到实际名称，即为

```cpp
MetadataOffset<const char*>(s_GlobalMetadata, s_GlobalMetadataHeader->stringOffset, index)
```

函数名获取的还是比较复杂的，最大的问题是入口太多，并且大部分都不是从IL2CPP转换的C#中来的

至此大概了解了il2cpp中gameassembly与global-metadata.dat的交互逻辑，后面详细学习一下官方文档全面理解il2cpp

### 官方对il2cpp的文章

官方在il2cpp发布时介绍了一系列il2cpp的设计理念以及具体的细节，可以先读此类内容再进行详细分析,也可以找下有无翻译

[The future of scripting in Unity](https://blog.unity.com/technology/the-future-of-scripting-in-unity)

[An introduction to IL2CPP internals](https://blog.unity.com/technology/an-introduction-to-ilcpp-internals)

### 知乎的翻译

[Unity将来时：IL2CPP是什么？](https://zhuanlan.zhihu.com/p/19972689)

[Unity将来时：IL2CPP怎么用？](https://zhuanlan.zhihu.com/p/19972666)

[用Unity做游戏，你需要深入了解一下IL2CPP](https://zhuanlan.zhihu.com/p/20029506)

[IL2CPP 深入讲解：代码生成之旅](https://zhuanlan.zhihu.com/p/20063880)

[Unity深入讲解：方法调用介绍](https://zhuanlan.zhihu.com/p/20167569)

[IL2CPP 深入讲解：泛型共享](https://zhuanlan.zhihu.com/p/20207712)

[IL2CPP 深入讲解：P/Invoke封装](https://www.jianshu.com/p/2fcbc313cf5c)

[IL2CPP 深入讲解：垃圾回收器的集成](https://www.jianshu.com/p/59d3ab1135b3)
