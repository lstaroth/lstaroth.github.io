UnKnoWnCheaTs分享过一个x64伪造函数调用栈/返回地址的文章
针对其原理做个简要分析，修复部分使用问题，以及重构了个简单版本的代码(增加少量代码执行量简化代码逻辑)
前置知识：栈回溯原理
原始代码地址：[x64 return address spoofing](https://www.unknowncheats.me/forum/anti-cheat-bypass/268039-x64-return-address-spoofing-source-explanation.html)
改进后的代码地址：[spoof_call](https://github.com/lstaroth/spoof_call)

### 原始spoof_call实现逻辑分析
先总结其核心原理，方便后面看代码：使用rbx非易失性寄存器保存返回地址，并找到一个gadget作为“伪造的返回地址”
先看其第一层函数的实现

```cpp
template <typename Ret, typename... Args>
static inline auto spoof_call(
	const void* trampoline,
	Ret(*fn)(Args...),
	Args... args
) -> Ret
{
	struct shell_params
	{
		const void* trampoline;
		void* function;
		void* rdx;
	};

	shell_params p{ trampoline, reinterpret_cast<void*>(fn) };
	using mapper = detail::argument_remapper<sizeof...(Args), void>;
	return mapper::template do_call<Ret, Args...>((const void*)&detail::_spoofer_stub, &p, args...);
}

//demo
int testfunction(int a,int b,int c,int d,int e){return a + b + c + d + e};
int res = spoof_call(&gadget,testfunction,1,2,3,4,5);
```

这里用模板萃取做了个函数转发,但是没处理好右值的问题以及不支持强制类型转换的问题(后续会提到怎么解决)
spoof_call函数内部调用do_call函数，模板参数是Ret, Args...，实际参数是shellcode地址，shell_params对象地址和展开的实际参数包。另外这里过早引入_spoofer_stub地址作为参数没有必要，步骤可以后移
再看下do_call的实现

```cpp
template <std::size_t Argc, typename>
struct argument_remapper
{
	// At least 5 params
	template <typename Ret, typename First, typename Second,
			  typename Third, typename Fourth, typename... Pack>
	static auto do_call(const void *shell, void *shell_param, First first, Second second,
						Third third, Fourth fourth, Pack... pack) -> Ret
	{
		return shellcode_stub_helper<Ret, First, Second, Third, Fourth, void *, void *, Pack...>(shell, first, second, third, fourth, shell_param, nullptr, pack...);
	}
};

template <std::size_t Argc>
struct argument_remapper<Argc, std::enable_if_t<Argc <= 4>>
{
	// 4 or less params
	template <typename Ret, typename First = void *, typename Second = void *,
			  typename Third = void *, typename Fourth = void *>
	static auto do_call(const void *shell, void *shell_param, First first = First{},
						Second second = Second{}, Third third = Third{}, Fourth fourth = Fourth{}) -> Ret
	{
		return shellcode_stub_helper<Ret, First, Second, Third,
									 Fourth, void *, void *>(shell, first, second, third, fourth, shell_param, nullptr);
	}
};
```

依然是一层函数参数转发，和std::function的设计差不多，不过在这里做了特化处理
参数个数大于4的实际调用函数，会去实例化并调用：3 + argc 个参数的 shellcode_stub_helper模板函数
参数个数等于4的实际调用函数，会去实例化并调用：3 + 4个参数的shellcode_stub_helper模板函数
也就是如果参数个数不满4个，那么会默认用0初始化伪造的函数做个兼容层，但实际上后者是固定参数的函数，完全没必要用模板，徒增理解难度，后面代码可以给他改掉
另外看下shellcode_stub_helper函数实现

```cpp
extern "C" void* _spoofer_stub();

	template <typename Ret, typename... Args>
	static inline auto shellcode_stub_helper(
		const void* shell,
		Args... args
	) -> Ret
	{
		auto fn = (Ret(*)(Args...))(shell);
		return fn(args...);
	}
```

就是第一个参数为shellcode地址，展开后续参数包调用进入shellcode内，下面看下shellcode的设计

```x86asm
_spoofer_stub PROC
    pop r11				;r11 == shellcode_stub_helper 的 ret addr
    add rsp, 8			;加完之后rsp指向reversed的args1
    mov rax, [rsp + 24]	;mov rax,&shell_param
        
    mov r10, [rax]		;mov r10,shell_param.trampoline
    mov [rsp], r10		;mov [rsp], shell_param.trampoline
        
    mov r10, [rax + 8]	;mov r10,shell_param.function
    mov [rax + 8], r11	;mov shell_param.function, shellcode_stub_helper 的 ret addr
     
    mov [rax + 16], rbx	;mov shell_param.rbx,rbx
    lea rbx, fixup		;rbx = &fixup
    mov [rax], rbx		;shell_param.trampoline = &fixup
    mov rbx, rax		; rbx = &shell_param
        
    jmp r10				; jmp org:shell_param.function
     
fixup:
    sub rsp, 16			;抬出shell_param + nullptr的空间
    mov rcx, rbx		;mov rcx,&shell_param
    mov rbx, [rcx + 16]	;mov rbx,shell_param.rdx(使用rdx前保存的值)
    jmp QWORD PTR [rcx + 8]	;jmp shell_param.function即shellcode_stub_helper 的 ret addr
_spoofer_stub ENDP
```

配合注释理解下即可，进入_spoofer_stub后的调用栈为
RSP-> shellcode_stub_helper调用shellcode的返回地址
arg1
arg2
arg3
arg4
shell_param
nullptr
arg5(if have)
arg6(if have)
...
这里实际上前4个参数的寄存器值是对的，rcx = arg1，rdx = arg2......
在jmp 10进入目标函数前确保RSP指向当前arg2的位置以保证目标函数对arg5的访问正常，同时修改arg2为gadget的地址

### 做一些spoof_call设计上的猜想解答,并不一定是作者实际想法
1. 为什么shellcode_stub_helper要写到struct argument_remapper的外面而不作为成员函数
因为他们俩的变参实际不一样，shellcode_stub_helper的变参要比后者多shell_param和nullptr所以没法混用
2. 为什么要扭曲到先传4个参数再传递自己的shell_param和nullptr再传剩余的参数
寄存器的值是正确的不用再额外修正了(为什么要先传前4个参数)
避免shellcode中因为不知道实际调用进来的参数个数从而不知道shell_param在栈中的位置(为什么shell_param不能放到最后)

### 如何解决不支持强制类型转换的问题
举例如下

```cpp
//demo
int testfunction1(int a,int b,int c,int d,int e){return a + b + c + d + e};
int res = spoof_call(&gadget,testfunction1,1,2,3,4,5);	//正常

int64_t testfunction2(int64_t a,int64_t b,int64_t c,int64_t d,int64_t e){return a + b + c + d + e};
int64_t res = spoof_call(&gadget,testfunction2,1,2,3,4,5);	//报错
```

testfunction1正常编译，但是testfunction2的调用会报错，原因是这么设计代码会让编译期根据实际传递的参数推测要调用的函数参数类型，然后去匹配，1,2,3,4,5这里的参数类型默认是int，编译器就去检测，那你传给我的testfunction2是接受5个int类型参数的函数吗，发现不是，实际是接受5个int64_t类型参数的函数，所以会报错，咋解决呢，就是不能让函数定义参数类型在实际传参的时候决定，提前到结构体构造中解决问题

```cpp
template<typename T>	concept NOT_USE_XMM = !std::is_floating_point_v<T>;

template <typename RET, typename... ARGS>
struct FunctionTraits {
};

template <typename RET, typename... ARGS>
struct FunctionTraits<RET(ARGS...)> {
};

template <typename RET, NOT_USE_XMM... ARGS>
struct FunctionTraits<RET(ARGS...)> {
	static inline RET spoof_call(const void* gadget, RET(*fn)(ARGS...), ARGS... args)
	{
		shell_params param{ .gadget = gadget, .realfunction_addr = fn };
		return argument_remapper<RET, ARGS...>::do_call(&param, args...);
	}
};

FunctionTraits<decltype(testfunc)>.spoof_call(gadget, &testfunc, 1, 2, 3, 4, 5, 6);
```

注：这里用了C++20的concept是为了简化代码，实际可以用std::enable_if替代实现
调用spoof_call实际传递参数之前的构造FunctionTraits结构体的时候就匹配好了ARGS参数类型，所以也就不会有上述的问题存在了，可以支持强制类型转换，用起来和正常函数差不多(用宏定义包装一下的话)，同时由于不支持浮点数参数类型，所以在编译的时候直接报错，免得运行时再崩溃，实际上去做下特化可以支持浮点数的类型，后续有时间再更新

### 优化后的代码逻辑
多执行点代码量，大幅简化了代码逻辑，详细demo见github页面
1. 避免实际参数被打断不连续传递
2. if constexpr替代特化做分支选择
3. 固定参数的shellcode_stub直接不做模板函数
4. 固定参数的默认构造函数兼容层在shellcode_stub_arg6中简化实现
5. shellcode中的返回地址不再复用
6. 解决不支持右值引用参数的问题

```cpp
namespace detail
{
	struct shell_params
	{
		const void* gadget;
		void* realfunction_addr;
		void* Nonvolatile_register;
		void* shellcode_retaddr;
		void* shellcode_fixstack;
	};
	
	template <typename RET>
	static inline RET shellcode_stub_arg6(const void* shell, void* shell_param, void* first = 0, void* second = 0, void* third = 0, void* forth = 0)
	{
		auto fn = (RET(*)(void*, void*, void*, void*, void*))(shell);
		return fn(shell_param, first, second, third, forth);
	}

	template <typename RET, typename... ARGS>
	static inline RET shellcode_stub(const void* shell, void* shell_param, ARGS&&... args)
	{
		auto fn = (RET(*)(void*, ARGS...))(shell);
		return fn(shell_param, std::forward<ARGS>(args)...);
	}

	template<typename RET, typename... ARGS>
	struct argument_remapper
	{
		static RET do_call(void* shell_param, ARGS&&... args)
		{
			if constexpr (sizeof...(args) >= 4)
			{
				return shellcode_stub<RET, ARGS...>(&NoStackShellcode, shell_param, std::forward<ARGS>(args)...);
			}
			else
			{
				return shellcode_stub_arg6<RET>(&NoStackShellcode, shell_param, (void*)std::forward<ARGS>(args)...);
			}
		}
	};

	template<typename T>	concept NOT_USE_XMM = !std::is_floating_point_v<T>;

	template <typename RET, typename... ARGS>
	struct FunctionTraits {
	};

	template <typename RET, typename... ARGS>
	struct FunctionTraits<RET(ARGS...)> {
	};

	template <typename RET, NOT_USE_XMM... ARGS>
	struct FunctionTraits<RET(ARGS...)> {
		static inline RET spoof_call(const void* gadget, RET(*fn)(ARGS...), ARGS&&... args)
		{
			shell_params param{ .gadget = gadget, .realfunction_addr = fn };
			return argument_remapper<RET, ARGS...>::do_call(&param, std::forward<ARGS>(args)...);
		}
	};
}
```

汇编代码修改如下，附带注释便于理解

```x86asm

.CODE
     
NoStackShellcode PROC
    mov [rsp + 8],  rcx     ;shell_param
    mov [rsp + 16], rdx     ;real_arg0
    mov [rsp + 24], r8      ;real_arg1
    mov [rsp + 32], r9      ;real_arg2

    ;shell_param成员修改
    mov rax, [rsp + 8]      ;rax = shell_param
    mov r10, [rsp]          ;r10 = _spoofer_stub函数的原始返回地址
    mov [rax + 24], r10     ;shell_param.shellcode_retaddr = _spoofer_stub ret addr
    mov [rax + 16], rbx     ;保存用于中转gadget使用的rbx的原始值到shell_param.rbx
    lea rbx, fixup          
    mov [rax + 32], rbx     ;shell_param.shellcode_fixstack = fixup
    lea rbx, [rax + 32]     ;rbx = &shell_param.shellcode_fixstack

    ;gadget地址写入栈返回地址
    mov r10, [rax + 0]      ;r10 = shell_param.trampoline = jmp_rbx_0
    mov [rsp + 8],r10       ;中转gadget写入返回地址 = jmp_rbx_0
    
    ;修复调用栈 + jmp realfunction
    add rsp, 8              ;rsp增加已跳过_spoofer_stub比目标function多一个参数对栈的影响(shell_param)
    mov rcx, [rsp + 8]      ;修复rcx寄存器传参
    mov rdx, [rsp + 16]     ;修复rdx寄存器传参
    mov r8, [rsp + 24]      ;修复r8寄存器传参
    mov r9, [rsp + 32]      ;修复r9寄存器传参
    mov r10, [rax + 8]      ;取出realfunction = shell_param.function
    jmp r10
     
fixup:
    sub rsp, 8              ;恢复add rsp, 8影响的堆栈
    lea rcx, [rbx - 32]     ;rcx = shell_param
    mov rbx, [rcx + 16]     ;rbx = shell_param.rbx
    push [rcx + 24]         ;push shell_param.shellcode_retaddr
    ret
NoStackShellcode ENDP

END
```
