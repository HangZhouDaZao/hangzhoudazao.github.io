---
layout: post
title: "C++ 动态加载共享库"
description: "c++ 动态加载"
category: "cpp"
tags: [cpp,programming,dlopen]
---
{% include JB/setup %}


* **注意:** 本节的所有叙述都基于Linux平台
* **参考文献:** [C++ dlopen](http://www.tldp.org/HOWTO/C++-dlopen/)
* **示例代码:** [dlopen demo1](https://github.com/fengya90/fsrc1/tree/master/dlopen_demo1)


## 问题

dlclose, dlerror, dlopen 和 dlsym这几个函数用于动态加载共享库，使得程序可以不在启动时加载所有的库，而是根据运行时的实际情况有选择的加载。

这套API在C语言里使用很方便，但是在cpp中使用却有点麻烦，主要问题在于cpp的两个特性：

1. 名字重整(name mangling)
2. 类(class)

C语言的全局函数（这里指non-static functions）调用的过程需要依靠PLT/GOT，在初始化过程中依赖符号表查找，如果是动态加载共享库后dlsym，也依赖于符号表。而使用C语言编写的共享库中的符号名和代码中的符号名是一致的。但是c++因为需要支持函数重载等一些C没有的特性，必须有name mangling，因为符号表中一个名字对应一个函数，但是代码中一个名字却有多个函数，这样，调用者无法像在C语言中直接使用dlsym,因为调用者并不能知道他需要的函数在符号表中到底是什么名字。

dlopen这套API原生只能支持调用函数，如何创建类又是另一个问题。

## 解决方式

简单的说通过extern "C"来解决名字重整问题，使用多态（polymorphism）来解决Class的问题。具体而言：

* 对动态库中的全局函数，即需要被dlsym的搜索的函数标志为extern "C"
* 动态库中每个类都对应一个全局函数，用于创建和销毁。创建类时返回基类，调用者通过虚函数执行功能。

下面以一个例子解释。

<?prettify?>
	#+NAME: speak.h
	#ifndef SPEAK_H
	#define SPEAK_H
	
	class Speak {
	public:
	    Speak(){}
	    virtual ~Speak() {}
	    virtual void Say() = 0;
	};
	
	typedef Speak* CreateSpeakFun();
	typedef void DestroySpeakFun(Speak*);
	
	#endif

首先创建一个基类Speak，后面所有的类继承Speak且实现功能函数Say。调用者调用虚函数Say。

下面创建一个库libspeak

<?prettify?>
	#include "speak.h"
	#include <iostream>
	using namespace std;
	
	class CatSpeak : public Speak {
	public:
		virtual void Say(){
			cout<<"cat"<<endl;
		}
	};
	
	extern "C" Speak* CreateCat() {
	    return new CatSpeak;
	}
	
	extern "C" void DestroyCat(Speak* p) {
	    delete p;
	}
	
	class DogSpeak : public Speak {
	public:
		virtual void Say(){
			cout<<"dog"<<endl;
		}
	};
	
	
	extern "C" Speak* CreateDog() {
	    return new DogSpeak;
	}
	
	extern "C" void DestroyDog(Speak* p) {
	    delete p;
	}


应用程序创建对象时用CreateCat和CreateDog，销毁时用DestroyCat和DestroyDog，且这些函数用extern "C" 修饰。代码编译，生成共享库。

下面就是应用程序怎么调用了，


<?prettify?>
	#include "speak.h"
	#include <dlfcn.h>
	#include <iostream>
	
	using namespace std;
	
	int main() {
		CreateSpeakFun*  create_fun;
		DestroySpeakFun* destroy_fun;
		const char* dlsym_error;
		Speak *s;
	
	#ifdef __APPLE__
		void* lib = dlopen("build/lib/libspeak.dylib", RTLD_LAZY);
	#elif __LINUX__
		void* lib = dlopen("build/lib/libspeak.so", RTLD_LAZY);
	#endif
		if (!lib) {
				cerr<<"open library error!"<<endl;
	            return -1;
	    }
	
	    dlerror(); //reset dlerror
	
	    create_fun = (CreateSpeakFun*) dlsym(lib, "CreateCat");
	    dlsym_error = dlerror();
	    if (dlsym_error) {
	    	cerr<<"search CreateCat failed!"<<endl;
	    	dlclose(lib);
	        return -2;
	    }
	
	    destroy_fun = (DestroySpeakFun*) dlsym(lib, "DestroyCat");
	    dlsym_error = dlerror();
	     if (dlsym_error) {
	    	cerr<<"search DestroyCat failed!"<<endl;
	    	dlclose(lib);
	        return -2;
	    }
	
	    s = create_fun();
	    s->Say();
	    destroy_fun(s);
	
	    create_fun = (CreateSpeakFun*) dlsym(lib, "CreateDog");
	    dlsym_error = dlerror();
	    if (dlsym_error) {
	    	cerr<<"search CreateDog failed!"<<endl;
	    	dlclose(lib);
	        return -2;
	    }
	
	    destroy_fun = (DestroySpeakFun*) dlsym(lib, "DestroyDog");
	    dlsym_error = dlerror();
	     if (dlsym_error) {
	    	cerr<<"search DestroyDog failed!"<<endl;
	    	dlclose(lib);
	        return -2;
	    }
	
	    s = create_fun();
	    s->Say();
	    destroy_fun(s);
	
		dlclose(lib);
	
	    return 0;
	}


## 使用场景

- 插件机制，写框架
- 热加载，经常变更的业务代码可以抽取出来，特定场景下可以做到不停机更新代码。
