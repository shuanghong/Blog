---
title: C++11 之 Universal 引用、完美转发
date: 2018-8-05 21:00:00 
categories: [C++, 右值引用]
toc: true
---

## Universal reference
上节提到 C++11 以后, 类型T的右值引用表示为 T&&, 然而并不是所有的形如 T&& 的类型都是右值引用, 有两种情况: 一种是普通的 rvalue reference, 另一种称为 universal reference(万能引用). 它既可能表示 lvalue reference, 也可能表示 rvalue reference.

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
	
	还有就是 auto, 如果一个对象的类型被定义为auto&&, 则此对象为 universal reference. 如前面的例子.

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

universal reference 即可以是 lvalue reference, 也可以是 rvalue reference,  那么如何确定 universal reference 的引用类型呢? 规则如下:

* 如果函数模板传入参数类型是左值, 那么 T&& 中的 T 将被推导成 T&, T&& 就是一个 lvalue reference.
* 如果函数模板传入参数类型是右值, 那么 T&& 中的 T 将被推导成原生类型, T&& 就是一个 rvalue reference.

这里涉及到了编译器的 reference collapsing(引用折叠)规则

### References Collasping

当编译器推导模板类型时, 如果出现 "对引用的引用"(reference to reference), 就会将自动将那些 references "合并"起来. 比如传入的参数是左值, T 被推导成int&, 那么 T&& 就会变成 int& &&, 这个时候就会发生 references collasping, 使得

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

T&& 作为 universal reference时, 既能绑定到左值(就像左值引用)上去, 也能绑定到右值(就像右值引用)上去. 既能绑定到 const 或者 非const 对象上去, 也能绑定到volatile 或 非volatile 对象上去, 甚至能绑定到 const volatile 对象上去, 它能绑定到几乎任何东西上去.

参考:  
[http://www.fastdrivers.org/zh-cn/archive/cpp-universal-ref-move-zh-cn.html](http://www.fastdrivers.org/zh-cn/archive/cpp-universal-ref-move-zh-cn.html)  
[https://www.sczyh30.com/posts/C-C/cpp-move-semantic/](https://www.sczyh30.com/posts/C-C/cpp-move-semantic/)  
[http://www.cnblogs.com/boydfd/p/5251797.html](http://www.cnblogs.com/boydfd/p/5251797.html)
