## std::move 使用场景

* STL 容器的相关函数如 vector::push_back, 会对参数对象进行复制, 这就会造成对象内存的额外创建. 原意是想把参数 push_back 进去就行了. 如下代码:

		int main(void) 
		{
		    std::string str = "Hello";
		    std::vector<std::string> v;
		    
		    v.push_back(str);   // call  push_back(const T&) 
		    std::cout << "After copy, str is \"" << str << "\"\n";
		
		    v.push_back(std::move(str));    //rvalue reference, push_back(T&&)
		    std::cout << "After move, str is " << str << std::endl;
		    std::cout << "The contents of the vector are " << v[0] << "," << v[1] << std::endl;
		}

* 参数类型是 std::unique_ptr. 

std::unique_ptr 具有对其所管理的对象资源的独占性, 不允许拷贝 std::unique_ptr. 即不能调用拷贝构造函数、在参数传递时不能按值传递 unique_ptr、不能将其传给需要拷贝副本的STL算法使用. 所以在这三个场景下都需要使用 std::move() 来转移所有权. 如下代码:

	#include <memory>
	#include <vector>
	class Socket
	{
	public:
	    explicit Socket(int sockfd)
	    :m_sockfd(sockfd)
	    {
	    }
	    ~Socket()
	    {
	    } 
	private:
	    int m_sockfd;
	};
	
	int main()
	{  
	    std::unique_ptr<Socket> socket1(new Socket(0));  
	    std::vector<std::unique_ptr<Socket>> sockets;  
	    sockets.push_back(socket1);  // 发生了 std::unique_ptr 的拷贝
	    return 0;
	}

编译器错误提示如下:

	/usr/local/include/c++/5.2.0/bits/unique_ptr.h:356:7: note: declared here
	       unique_ptr(const unique_ptr&) = delete;

将 main() 修改如下则可以编译通过.

	int main()
	{  
	    std::unique_ptr<Socket> socket1(new Socket(0));  
	    std::vector<std::unique_ptr<Socket>> sockets;  
	    sockets.push_back(std::move(socket1));  // 触发移动语义, 转移了 socket1 的所有权
	    return 0;
	}

同样也可以修改如下:

	int main()
	{  
	    std::vector<std::unique_ptr<Socket>> sockets;  
	    sockets.push_back( std::unique_ptr<Socket> (new Socket(0)) ); // 匿名临时变量也是右值, 调用 std::unique_ptr 的移动构造函数
	    return 0;
	}

std::unique_ptr 参数传递时也不能按值传递. 代码如下:

	#include <memory>
	#include <vector>
	#include <iostream>
	
	class Socket
	{
	public:
	    explicit Socket(int sockfd)
	    :m_sockfd(sockfd)
	    {
	    }
	    ~Socket()
	    {
	    }
	    
	    int getFd(){return m_sockfd;}
	    
	private:
	    int m_sockfd;
	};
	
	int getSocketFd(std::unique_ptr<Socket> socket)
	{
	    return socket->getFd();
	}
	
	int main()
	{  
	    std::unique_ptr<Socket> socket1 = std::make_unique<Socket>(0);  
	    int fd = getSocketFd(socket1);
	    std::cout << fd << std::endl;
	    return 0;
	}


编译器错误提示如下:

	main.cpp:30:33: error: use of deleted function 'std::unique_ptr<_Tp, _Dp>::unique_ptr(const std::unique_ptr<_Tp, _Dp>&) [with _Tp = Socket; _Dp = std::default_delete<Socket>]'
	     int fd = getSocketFd(socket1); 
	In file included from /usr/local/include/c++/5.2.0/memory:81:0,
                 from main.cpp:1:
	/usr/local/include/c++/5.2.0/bits/unique_ptr.h:356:7: note: declared here
	       unique_ptr(const unique_ptr&) = delete;

修改如下:

	int getSocketFd(std::unique_ptr<Socket> socket)
	{
	    std::unique_ptr<Socket> skt = std::move(socket);
	    return skt->getFd();
	}
	
	int main()
	{  
	    std::unique_ptr<Socket> socket1 = std::make_unique<Socket>(0);  
	    int fd1 = getSocketFd(std::move(socket1));
	
	    std::cout << fd1 << std::endl;
	    return 0;
	}  