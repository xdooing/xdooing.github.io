---
title: 多线程环境下std vector重新申请空间导致的内存重复释放
date: 2024-11-18 00:00:00 +0800
categories: [Debug]
tags: [多线程]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---





> 最近遇到了一个bug，自己第一时间没有注意到（但是manager看一眼就注意到了），因此在这里记录一下，加深印象



## 具体问题

首先说说具体情况，原本一个函数一直都没出问题，最近新添加了一个case之后，会在一处delete指针的地方crash，报错invalid pointer，下面是代码：

```c++
struct ThreadParam {
	int* ptr_;
};

void threadFunc(void* param) {
	
	/*...*/
	
	// 线程内release memory
	if(param->ptr_) {
		delete param->ptr_;
		param->ptr_ = nullptr;
	}
}

void mainFunc() {
	
	// 用于线程task的params
	std::vector<ThreadParam> params;
	params.reserve(256);
	
	// for 循环创建线程任务
	int count = 0;
	for(/*....*/) {
		ThreadParam param;
		param.ptr_ = new int(10);
		params.push_back(param);
		
		thread->add(threadFunc, &params[count]);
		count++;
	}
	
	// 等待所有thread执行完毕
	thread->waitAllJobs();
	
	// 检查内存是否释放完毕
	for(int i = 0; i < count; ++i) {
		auto* ptr = params[i].ptr_;
		if(ptr) {
			delete ptr;
			params[i].ptr_
		}
	
	}
}
```

这个函数最后会在mainFunc中`delete ptr`处crash，原因是invalid pointer。

其实问题就出在 `params.reserve(256)` 以及 `params.push_back(param)`这两处地方，std vector在进行push_back的时候，如果空间不够，会重新分配一块更大的连续内存空间，将原来的元素复制（或移动）到新的内存空间中，然后再将新元素添加到新内存空间的末尾，最后释放原来的内存空间。

造成的结果就是，线程内释放了ptr_，所有线程执行结束之后，主线程执行`auto* ptr = params[i].ptr_`，拿到的ptr并不是被线程置为nullptr的指针，而是std vector扩容后拷贝过去的，所以它并不为空，但是指向的内存是已经被释放了的，最后造成了crash。

至于为什么之前的case并没有报错，原因是 `params.reserve(256)` ，之前的case没有进行扩容。



## 解决方法

将`std::vector<ThreadParam> params`改为`std::vector<ThreadParam*> params`即可解决。







