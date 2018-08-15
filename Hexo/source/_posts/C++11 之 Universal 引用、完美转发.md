---
title: C++11 之 Universal 引用、完美转发
date: 2018-8-05 21:00:00 
categories: [C++, 右值引用]
toc: true
---

## Universal reference
[上节](https://shs19890608.github.io/2018/04/01/C++11%20%E4%B9%8B%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8%E3%80%81%E7%A7%BB%E5%8A%A8%E8%AF%AD%E4%B9%89/)提到 C++11 以后, 类型T的右值引用表示为 T&&, 然而并不是所有的形如 T&& 的类型都是右值引用, 有两种情况: 一种是普通的 rvalue reference, 另一种称为 universal reference(万能引用). 它既可能表示 lvalue reference, 也可能表示 rvalue reference.

	Widget&& var1 = someWidget;  // var1 是个普通的右值引用
	auto&& var3 = var1;  //var3 是个universal reference, 由于var1 是左值,所以推导结果是左值引用
	auto&& var2 = 10   //var2 是个universal reference, 由于 10 是个右值，所以推导结果是右值引用

	template<typename T>
	void f(T&& param);    // “T&&” 意味着 universal reference
	​
	f(10);   // 10 是右值，所以 T&& 最终变成右值引用，也就是 int&&

### Universal reference 判定
判断是否为 universal reference, 主要根据以下两点:

* 类型必须为 T&&, 加上 const 修饰就不是 universal reference 了, 比如下面的 参数类型 T&& 为 rvalue reference 而不是 universal reference

		template<typename T>
		void f(const T&& param);

* 存在类型推导, 即 T&& 中 T 是需要推导的类型.  
	常用的情况是在函数模板中, 如
 
		template<typename T>
		void f(T&& param);              // param 类型是一个 universal reference
	
	还有就是 auto, 如果一个对象的类型被定义为auto&&, 则此对象为 universal reference, 如前面例子以及下面的代码

		int i = 0;
		
		// int&& j = i;	error: cannot bind rvalue reference of type 'int&&' to lvalue of type 'int'
		auto&& j = i;	// universal reference 不同于普通的右值引用, 可以绑定到左值
		auto&& k = 0;	// universal reference 也可以绑定到右值

类型为 T&& 以及参与类型推导缺一不可, 如以下 vector 类中的 push_back 函数, 虽然参数类型为T&&，但是并未参与类型推导. vector 模板实例化的时候, T的类型已经被推导. 当编译器再去定义push_back() 时, T 已经被推导出来了, 因此 T&& 是 rvalue reference.

	template<class T, class Allocator = allocator<T>>
	class vector {
	public:
		void push_back(T&& x);
		// ...
	};

	std::vector<Widget> v;	// vector 模板被实例化如下
	
	class vector<Widget, allocator<Widget>> {
	public:
	    void push_back(Widget&& x);     //右值引用
	    ...
	};

而 emplace_back 模板函数则需要进行类型推导, 类型参数 Args 独立于 vector 的类型参数T, 所以每次 emplace_back 被调用的时候, Args 必须被推导. 因此对应的参数 Args&& 为 universal reference

	template<class T, class Allocator = allocator<T>>
	class vector {
	public:
	    template <class... Args>
	    void emplace_back(Args&&... args);
	    ...
	};

ps. emplace_back的类型参数被命名为 Args 而不是 T, 我们前面说到的类型必须为 T&&, 这里的 T 只是一个类型名字, 也可以是任意其他的名字, 如 

	template<typename MyTemplateType>
    void Func(MyTemplateType&& param); // param是一个 universal reference

### Template 类型推导
如下函数模板定义

	template<typename T>
	void f(ParamType param);

	f(expr);

编译器根据 expr 来推导两个类型: T 和 ParamType, 通常这两个类型是不一样的, ParamType 常常包含一些修饰符, 如下:

	template<typename T>
	void f(const T& param);

编译器对 T 类型推导时, 不仅仅取决于调用时传入的 expr 类型, 同时取决于 ParamType, 有3种不同的情况:

* ParamType 是指针或引用, 但不是一个universal引用
* ParamType 是 universal 引用
* ParamType 不是指针也不是引用

当 ParamType 是 universal 引用时, 规则如下:

* 如果 expr 是一个左值, 则 T 会被推导成左值引用, ParamType 也会被推导成左值引用(采用引用折叠规则), 虽然在语法上被声明成一个右值引用
* 如果 expr 是一个右值, 则 T 的类型与 expr 类型相同

参考 [http://https://www.cnblogs.com/boydfd/p/4950334.html](http://https://www.cnblogs.com/boydfd/p/4950334.html)

### References Collasping

当编译器推导模板类型时, 如果出现 "对引用的引用"(reference to reference), 就会将自动将那些 references "合并"起来. ParamType 是 universal reference 时,调用传入左值, T 被推导成int&, 那么 T&& 就会变成 int& &&, 这个时候就会发生 references collasping, 使得

	T&&  ⇒  int& &&  ⇒  int&

References collasping 的规则如下:

* T& &  ⇒ T&
* T& && ⇒ T&
* T&& & ⇒ T&
* T&& && ⇒ T&&

如果两个引用中至少其中一个引用是左值引用, 那么折叠结果就是左值引用; 只有 "对右值引用的引用" 才会推导出右值引用, 其他的都会推导出左值引用.

	template<typename T>
	void f(T&& param);

	int x;​
	f(x);	// x 是一个左值, T 被推导为 T&, 则有 T&&  ⇒  int& &&  ⇒  int&, 左值引用
	f(10);	// 10 是一个右值, T 被推导为原生类型 int, 则有 T&&  ⇒  int &&  ⇒  int&&, 右值引用

如果代码变成如下形式:

	template<typename T>
	void f(T&& param);

	int x;
	int&& r1 = 10;
	int& r2 = x;
	​
	f(r1);	// 虽然r1, r2 一个是右值引用, 一个是左值引用, 但是由于r1 与 r2 都是左值, 所以最终得到的类型都是左值引用 int&.
	f(r2);

T&& 作为 universal reference 时, 既能绑定到左值(就像左值引用)上去, 也能绑定到右值(就像右值引用)上去. 既能绑定到 const 或者 非const 对象上去, 也能绑定到volatile 或 非volatile 对象上去, 甚至能绑定到 const volatile 对象上去, 它能绑定到几乎任何东西上去.

参考:  
[http://www.fastdrivers.org/zh-cn/archive/cpp-universal-ref-move-zh-cn.html](http://www.fastdrivers.org/zh-cn/archive/cpp-universal-ref-move-zh-cn.html)  
[https://www.sczyh30.com/posts/C-C/cpp-move-semantic/](https://www.sczyh30.com/posts/C-C/cpp-move-semantic/)  
[http://www.cnblogs.com/boydfd/p/5251797.html](http://www.cnblogs.com/boydfd/p/5251797.html)

## std::move 标准库实现

在 std::move() 的标准库实现中, 其参数类型就是 universal reference.

	template<teypename T>
	typename remove_reference<T>::type&& move(T&& t) noexcept
	{
		return static_cast<typename remove_reference<T>::type&&>(t);
	}

当调用时传递给 move(T&& t) 的实参为分别左值和右值时, 其推导过程不同.

	string s1("hi");
	string s2 = std::move(string("hi"));
	string s3 = std::move(s1);

在执行 s2 时, 流程如下:

* string("hi") 是右值, 根据 Template 推导规则, T 为 string 类型
* remove_reference 用 string 进行实例化, remove_reference<string>::type 是 string
* move() 返回类型是 string&&
* move() 参数 t 类型为 string&&

这个调用实例化为 string&& move(string&& t), 函数体返回 static_cast<string&&>(t), t 的类型本身为 string&&, static_cast 什么都不做, std::move() 的结果就是返回右值引用.

在执行 s3 时, 流程如下:

* s1 是左值, 根据 Template 推导规则, T 为 string&(string 引用)
* remove_reference 用 string& 进行实例化, remove_reference<string>::type 仍然是 string
* move() 返回类型是 string&&
* move() 参数 t 类型为 string& &&, 折叠为 string&

这个调用, 实例化的结果为: string&& move(string& t), 虽然 t 类型为 string&, 但是 static_cast 将其转换为 string&& 并返回.

因此, std::move() 的工作就是将传入的参数转换成其对应的右值引用类型.

## 完美转发(Perfect forwarding)

### 引入背景

有的时候, 我们需要将一个函数的参数原封不动地传递给另一个函数. 这里不仅需要参数的值不变, 而且需要参数的类型属性(左值、右值, const、volatile)保持不变, 这叫做 Perfect Forwarding(完美转发).

	template <typename T>
	void func(T t)
	{
	    cout << "func()" << endl;
	}
	
	template <typename T>
	void forwarding(T t)
	{
	    func(t);
	}
上面的代码中, forwarding 是一个转发函数模板, func 是真正执行代码的目标函数. 对于func 而言, 它总是希望转发函数将参数按照调用 forwarding 时的类型传递, 即传入  forwarding 的是左值对象, func 就能获得左值对象, 即传入 forwarding 的是右值对象, func 就能获得右值对象, 而不产生额外的开销, 就好像转发函数不存在一样.

上面的例子中, 函数 forwarding 中使用了最基本类型进行转发, 参数 t 在传给 func 时产生了一次额外的对象拷贝, 这样的转发只能说是正确的转发, 但是不完美.

为了避免拷贝, 我们需要将 forwarding 参数设为引用类型, 具体如下:

* 为了传递非 const 左值, forwarding  形参 t 必须为 T& 类型或者 const T& 类型
* 为了传递 const 左值, forwarding 形参 t 必须为 const T& 类型
* 为了传递非 const 右值, forwarding 形参 t 必须为 const T& 类型或者 T&& 类型或者 const T&& 类型
* 为了传递 const 右值, forwarding 形参 t 必须为 const T& 类型或者 const T&& 类型

似乎只要将 forwarding 参数类型设为 const T& 即可, [上节](https://shs19890608.github.io/2018/04/01/C++11%20%E4%B9%8B%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8%E3%80%81%E7%A7%BB%E5%8A%A8%E8%AF%AD%E4%B9%89/)也提到, const 左值引用可以绑定到任何类型.

	template <typename T>
	void func(T t)
	{
	    cout << "func()" << endl;
	}
	
	template <typename T>
	void forwarding(const T& t)
	{
	    func(t);
	}

但是当目标函数 func 的参数类型是一个普通的非 const 类型时, 无法接受 const 引用作为参数.

	void func(int t)
	{
	    t = ....
	}

并且当传入 forwarding 是一个右值时, 其函数体 func(t) 中的 t 却是一个具名的左值.

### 保持类型信息的函数参数
通过将转发函数 forwarding 的参数类型定义为 universal reference, 可以保持其对应实参的所有类型信息. 而使用引用参数(无论是左值还是右值)使得我们可以保持 const 属性, 因为在引用类型中的 const 是底层的, 如果我们把参数参数定义为 T1&& 和 T2&&, 通过引用折叠就可以保持转发实参的左值/右值属性.(摘自 C++ Ptimer 5th Chapter 16.2.7)

	template <typename T>
	void forwarding(T&& t)
	{
	    func(t);
	}

然而这样的 forwarding 并不能接受传入右值, 比如我们想要执行下面的目标函数, forwarding 在调用 func 时 t 变成了一个左值.

	void func(int&& t)
	{
	    std::cout << "int&&\n";
	}

因此我们需要在转发调用的时候保持 t 的类型属性.

### std::forward 实现完美转发
C++11 中使用 std::forward 实现了完美转发

	template <typename T>
	void forwarding(T&& t)
	{
	    func(std::forward<T>(t));
	}

通过下面的例子, 实现对不同参数类型的目标函数的调用.

	//void process_value(int i) { std::cout << "int\n"; }
	void func(int& i) { std::cout << "int&\n"; }
	void func(const int& i) { std::cout << "const int&\n"; }
	void func(int&& i) { std::cout << "int&&\n"; }
	void func(const int&& i) { std::cout << "const int&&\n"; }
	
	template <typename T>
	void forwarding(T&& t)
	{
		func(std::forward<T>(t));
	}
	 
	int main()
	{   
	    int a = 0;
	    forwarding(a);
	    
	    const int b = 0;
	    forwarding(b);
	    
	    forwarding(std::move(a));
	    forwarding(std::move(b));
	    
	    return 0;
	}
output:

	int&
	const int&
	int&&
	const int&&


