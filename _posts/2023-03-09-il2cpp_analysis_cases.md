### 背景
分享一些难点分析文章，大部分来源于katyscode的分享  


[Reverse Engineering Adventures: League of Legends Wild Rift (IL2CPP)](https://katyscode.wordpress.com/2021/01/15/reverse-engineering-adventures-league-of-legends-wild-rift-il2cpp/)

1. 分析并观察发现：Il2CppMetadataRegistration （以及 Il2CppCodeRegistration ）的字段顺序已被打乱
2. 发现字符串信息读取错误(非字面量)，通过最终null结尾的方式并加以xor运算符猜测得到解密xorkey，并且每个字符串都有自己的 XOR 密钥，通过上述方式自动化处理
3. 发现部分函数导出的api名称不对，一眼凯撒加密，但核心问题是为何加密后不影响正常的导出函数调用呢？

[Reverse Engineering Adventures: Honkai Impact 3rd (Houkai 3) (IL2CPP) (Part 1)](https://katyscode.wordpress.com/2021/01/17/reverse-engineering-adventures-honkai-impact-3rd-houkai-3-il2cpp-part-1/)

1. 小吹一波米忽悠，il2cpp加密的黄金标准！！
2. 采用眼神观察法看了一眼global-metadata确认虽然头部像是乱码，但是仔细一看0x40开始出现了很多低熵数据，因此至少不是全文件加密的形式
3. 用熵的形式去评估哪块内存有被加密似乎是个不错的技巧
4. 通过观察发现(666)每间隔0x353C0 字节长度的块之后的新快的前0x40字节都会被加密(刚好与2呼应)
5. 不知道加密方式是啥秘钥是啥怎么办，结合header头中未加密的偏移和长度对去计算metadata有效部分，从而反向去看是否存在无效部分(即可能的秘钥位置)，发现了0x4000的未知数据块，超出了元数据表
6. 找解密函数的位置通过ProcMon对metadata的访问下监控并查看堆栈，maybe ce更好？
7. 发现首先访问的是metadata的文件末尾那0x4000未知数据块的一部分
8. 并且根据经验可知metadata的读取是闭环在assembly中完成的，本身栈会到unityplayer中就是个很可疑的事
9. 逆向分析根据经验和关键的unmap函数调用确定LoadMetadataFile跳转到了unityplayer的某个函数中，并且在其中进行了解密逻辑
10. 明确这一点后把一段函数抠出来直接call调用或者模拟执行尝试解密

[Reverse Engineering Adventures: VMProtect Control Flow Obfuscation (Case study: string algorithm cryptanalysis in Honkai Impact 3rd)](https://katyscode.wordpress.com/2021/01/23/reverse-engineering-adventures-vmprotect-control-flow-obfuscation-case-study-string-algorithm-cryptanalysis-in-honkai-impact-3rd/)

1. 调试分析GetStringLiteralFromIndex
2. 由于主进程加了壳反调试因此通过loaddll的方式解决
3. 调用输出即为明文…
4. 后续进行控制流平坦化分析与算法的还原，并锐评了一下米忽悠的算法设计，表示这个算法考虑欠佳，如果采用密码学分析就可以出结果了(果然是大佬…)，这部分过于复杂没法总结

[IL2CPP Tutorial: Finding loaders for obfuscated global-metadata.dat files](https://katyscode.wordpress.com/2021/02/23/il2cpp-finding-obfuscated-global-metadata/)

1. 介绍找到关键函数LoadMetadataFile的思路
2. 通过调用链il2cpp_init-> il2cpp::vm::Runtime::Init-> il2cpp::vm::MetadataCache::Initialize-> il2cpp::vm::MetadataLoader::LoadMetadataFile
3. 而il2cpp_init一般是导出函数，分析对比跟踪定位关键函数的思路
4. 遇到il2cpp_init一般是导出函数做手脚的话去通过字符串找，这里思路根据加密方式的不同变化
5. 这里突然有一个灵感，打开思路，或许除了主动调用还有一个给到地址顺序写入的方式可以支持，这种方式似乎更少但需要考虑的是通过unityplayer反向定位下读写短点反推关键逻辑地址

[Reverse Engineering Adventures: Brute-force function search, or how to crack Genshin Impact with PowerShell](https://katyscode.wordpress.com/2021/01/24/reverse-engineering-adventures-brute-force-function-search-or-how-to-crack-genshin-impact-with-powershell/)

1. 思路是对加载到内存中的unityplayer进行fuzz调用，假定有一个函数形式为`typedef unsigned char* (__cdecl* DecryptDelegate)(unsigned char* encryptedData, int length);`
2. 从txt段一行行当做函数调用然后判断是否有对globalmetadata修改然后保存筛选判断  
3. 采用ida识别函数起始地址或许更快 ，但是会有混淆带来的识别率问题，可以配合idapython脚本实现加上unicorn，加入时间限制多线程支持后进行优化  

[IL2CPP Reverse Engineering Part 1: Hello World and the IL2CPP Toolchain](https://katyscode.wordpress.com/2020/06/24/il2cpp-part-1/)

1. 提供了一个Hello world级别的C#,IL,C++的对照代码

```CPP
#include <stdio.h>
 
int main(int argc, char **argv) {
    int a = 1;
    int b = 2;
    printf("Hello world: %d\r\n", a + b);
}
------------------------------------------
ldc.i4.1
stloc.0
ldc.i4.2
stloc.1
ldstr "Hello World: {0}"
ldloc.0
ldloc.1
add
box System.Int32
call System.Void System.Console::WriteLine(System.String,System.Object)
ret
------------------------------------------
IL2CPP_EXTERN_C IL2CPP_METHOD_ATTR void Program_Main_m7A2CC8035362C204637A882EDBDD0999B3D31776 (StringU5BU5D_t933FB07893230EA91C40FF900D5400665E87B14E* ___args0, const RuntimeMethod* method)
{
    static bool s_Il2CppMethodInitialized;
    if (!s_Il2CppMethodInitialized)
    {
        il2cpp_codegen_initialize_method (Program_Main_m7A2CC8035362C204637A882EDBDD0999B3D31776_MetadataUsageId);
        s_Il2CppMethodInitialized = true;
    }
    int32_t V_0 = 0;
    int32_t V_1 = 0;
    {
        V_0 = 2;
        int32_t L_0 = V_0;
        V_1 = ((int32_t)il2cpp_codegen_add((int32_t)1, (int32_t)L_0));
        int32_t L_1 = V_1;
        int32_t L_2 = L_1;
        RuntimeObject * L_3 = Box(Int32_t585191389E07734F19F3156FF88FB3EF4800D102_il2cpp_TypeInfo_var, &L_2);
        IL2CPP_RUNTIME_CLASS_INIT(Console_t5C8E87BA271B0DECA837A3BF9093AC3560DB3D5D_il2cpp_TypeInfo_var);
        Console_WriteLine_m22F0C6199F705AB340B551EA46D3DB63EE4C6C56(_stringLiteral331919585E3D6FC59F6389F88AE91D15E4D22DD4, L_3, /*hidden argument*/NULL);
        return;
    }
}
```