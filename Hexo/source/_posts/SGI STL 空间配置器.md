---
title: SGI STL 空间配置器
date: 2016-9-18 21:40:30 
categories: STL
toc: true
---

# 前言
Allocator 处于STL的底层, 为所有的容器提供空间分配和管理, 是其他组件的基石.  
下图(摘自《STL源码剖析》P6)显示了STL各组件之间的关系: Container 通过 Allocator 取得数据存储空间, Alogrithm 通过 Iterator 存取 Container 内容, Functor 可以协助 Algorithm 完成不同的策略变化, Adapter 可以修饰或套接 Functor.
![](http://i.imgur.com/BXiaYop.jpg)

# 简介
C++标准规范了 allocator 的一些必要接口，由各个厂家实现, SGI版本(V3.3)使用的是标准的std::allocator.  
PS. 《STL源码剖析》P47 说SGI STL的配置器与标准规范不同, 名称是 alloc 而非 allocator, 使用的话正确写法是  

	vector<int, std::allocator<int> > iv  
而不是  

	vector<int, std::alloc> iv  
**已经过时了.**  
参见 SGI V3.3 stl_vector.h line 154  

	template <class _Tp, class _Alloc = __STL_DEFAULT_ALLOCATOR(_Tp) >
	class vector : protected _Vector_base<_Tp, _Alloc> 
	{
		...
	}

以及 stl_config.h line 479

	# ifndef __STL_DEFAULT_ALLOCATOR
	#   ifdef __STL_USE_STD_ALLOCATORS
	#     define __STL_DEFAULT_ALLOCATOR(T) allocator< T >
	#   else
	#     define __STL_DEFAULT_ALLOCATOR(T) alloc
	#   endif
	# endif

同时也可以在[cpp reference](http://en.cppreference.com/w/cpp/container/vector) 上查看 vector 的定义
  
    template<
		class T,
		class Allocator = std::allocator<T>
	> class vector;

以及下面的demo code

	#include <iostream>
	#include <vector>
	
	int main(void)
	{
	    std::vector<int, std::allocator<int>> ivec; //不指定allocator时也是使用默认的Allocator = std::allocator<T>
	
	    ivec.push_back(3);
	    std::cout << ivec[0] << std::endl;
	
	    return 0;
	}

# overview
![](http://i.imgur.com/raRHSUE.jpg)
# allocator 定义
参见 stl_alloc.h line 588~628

    template <class _Tp>
	class allocator {
	  typedef alloc _Alloc;          // The underlying allocator.
	public:                          
	typedef size_t     size_type;
	...
	}

主要的成员函数有

	  // __n is permitted to be 0.  The C++ standard says nothing about what
	  // the return value is when __n == 0.

	  _Tp* allocate(size_type __n, const void* = 0) {
	    return __n != 0 ? static_cast<_Tp*>(_Alloc::allocate(__n * sizeof(_Tp))) 
	                    : 0;
	  }
	
	  // __p is not permitted to be a null pointer.
	  void deallocate(pointer __p, size_type __n)
	    { _Alloc::deallocate(__p, __n * sizeof(_Tp)); }
	
	  void construct(pointer __p, const _Tp& __val) { new(__p) _Tp(__val); }
	  void destroy(pointer __p) { __p->~_Tp(); }

其中内存分配函数 allocate() 调用 alloc::allocate(), 参数由分配的对象个数转为字节数, 而 alloc 或为第一级配置器 \__malloc\_alloc\_template (stl\_alloc.h line 188, line 250)

	typedef __malloc_alloc_template<0> malloc_alloc;
	typedef malloc_alloc alloc;

或为第二级配置器 \__default\_alloc\_template (stl\_alloc.h line 402)

	typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc;

# 配置器的实现
这里主要讲第二级配置器 \__default\_alloc\_template 且不考虑多线程的情况.

在 C++ 中一般使用 new/delete 进行内存分配/释放操作. 例如  

	class Foo { ... };
	Foo* pf = new Foo;
	...
	delete pf;
这两个操作都包含两个步骤:  
new 操作步骤  
1). 调用 ::operator new 配置内存  (关于 new, operator new, placement new 相关介绍, 参考 [这里](http://www.cnblogs.com/luxiaoxun/archive/2012/08/10/2631812.html))  
2). 调用构造函数 Foo::Foo() 构造对象并初始化  
delete 操作同理, 过程相反  
1). 调用析构函数 Foo::~Foo() 析构对象  
2). 调用 ::operator delete 释放内存

从上面 allocator的成员函数可以看出, 在 SGI 中, 将这两个步骤分开操作, 内存配置/释放由 alloc::allocate()/alloc::deallocate() 完成, 对象构造/析构由 ::construct()/::destroy() 完成. 目的是提高效率, 节省某些不必要调用构造函数的开销.  
![](http://i.imgur.com/g2671k7.jpg)

### \__default\_alloc\_template 定义
参见 stl_alloc.h line 289~400, 除了被 allocator 调用的 allocate()/deallocate()外, 还有以下关键成员

	  union _Obj {
	        union _Obj* _M_free_list_link;
	        char _M_client_data[1];    /* The client sees this.        */
	  };
	private:
	# if defined(__SUNPRO_CC) || defined(__GNUC__) || defined(__HP_aCC)
	    static _Obj* __STL_VOLATILE _S_free_list[]; 
	        // Specifying a size results in duplicate def for 4.1
	# else
	    static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS]; 
	# endif
	  static  size_t _S_freelist_index(size_t __bytes) {
	        return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);
	  }
	
	  // Returns an object of size __n, and optionally adds to size __n free list.
	  static void* _S_refill(size_t __n);
	  // Allocates a chunk for nobjs of size size.  nobjs may be reduced
	  // if it is inconvenient to allocate the requested number.
	  static char* _S_chunk_alloc(size_t __size, int& __nobjs);
	
	  // Chunk allocation state.
	  static char* _S_start_free;
	  static char* _S_end_free;
	  static size_t _S_heap_size;

### 内存池
考虑到小块内存分配导致的内存碎片问题, 二级分配器实现了一个轻量级的内存池, 并且考虑到在多线程环境下内存池的互斥访问问题.  
当申请内存小于128bytes时, 直接从内存池分配, 并维护 free list, 由链表来分配同样的小内存和回收小内存. 同时为了方便管理, 将任何小块内存申请量上调为8的倍数, 小块内存最大为 128bytes, 这样便需要 128/8 = 16 个 free list, 每个链表分别维护的区块大小为 8, 16, 24, 32...128 bytes.   
节点结构定义如下, 关于下面union类似的用法参考另外一篇博客.

    union obj
    {
        union obj *next;
        char data[1];
    };
链表定义如下  

    static obj *free_list[num_of_list_];
链表内存管理示意图
![](http://i.imgur.com/YfiImVG.jpg) 

简单的code实现  
![](http://i.imgur.com/r9F1GRa.jpg)

源码中 myString 的实现, 当执行 myString string1("C-style string1"); 时传入参数 15bytes, 实际申请 16 bytes X 20(块), 完成后交给 free\_list[1] 管理.  
当执行 myString string2("C-style string2")时, free\_list[1] 已有可用的内存空间. 下图中 free\_list 是一个二级指针, 0x804b0a0, +index 为 0x804b0a4, 该地址上的数据从内存图上可知为 0x0804c018, 即为 free\_list[1], 同时 result = \*my\_free\_list = 0x0804c018, 这就是返回给调用者的可用的内存空间.
![](http://i.imgur.com/4kHdUha.jpg)

从下图中可看出, 0x0804c018 上的值 next 为 0x0804c028,  0x0804c028 上的值为 0x0804c038...这样构成了list. 而已经使用的地址空间为 0x0804c008, 值为 C-style string1.
![](http://i.imgur.com/udeoDes.jpg)



### 成员函数, 空间分配与回收
- allocate() 空间分配  
流程图   
![](http://i.imgur.com/RgsXIvD.jpg)  

当free list有剩余空间时,获取内存块的操作示意图(摘自《STL源码剖析》P63 图2-5)  
![](http://i.imgur.com/WqO2GMA.jpg)

- refill() 填充自由链表  
流程图  
![](http://i.imgur.com/EdOBwIJ.jpg)

- allocChunk() 从内存池或系统heap或更大区块的 free list 取空间  
流程图
![](http://i.imgur.com/dODzEue.jpg)

内存池实际操作结果图(摘自《STL源码剖析》P69 图2-7)
![](http://i.imgur.com/OKro8SC.jpg)

- deallocate() 空间回收  
流程图  
![](http://i.imgur.com/w6ZU764.jpg)  
区块回收示意图(摘自《STL源码剖析》P65 图2-6)  
![](http://i.imgur.com/YWyWI1d.jpg)  

注: 从 deallocate() 实现来看, 回收空间时只是把空间交给 free list, 并没有还给系统 heap, 而在分配时如果内存池中的空间不够, 则会从系统 heap 分配空间(allocChunk).

### 成员函数, 构造与析构

	void construct(pointer __p, const _Tp& __val) { new(__p) _Tp(__val); }	// placement new, 调用构造函数 _Tp::_Tp(__val);
	void destroy(pointer __p) { __p->~_Tp(); }

虽然 allocator 定义了 construct()/destroy(), 但是如前面所说, 对象构造/析构实际上由全局函数 ::construct()/::destroy() 完成, 位于 stl_construct.h 中, 并结合其他3个全局函数 uninitialized\_copy()、uninitialized\_fill()、uninitialized\_fill\_n() 用来填充、复制大块内存数据, 位于 stl\_uninitialized.h 中.