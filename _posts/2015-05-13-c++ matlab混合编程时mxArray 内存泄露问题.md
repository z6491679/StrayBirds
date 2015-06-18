---
layout: post
title: c++ matlab混合编程时mxArray 内存泄露问题
category: 科研
comments: true
---


对于mxArray在创建完后若使用完一定要注意回收空间，不然会memory leak.如果是mxstruct，只要对其中的field进行了赋值操作，那么其原先的field一定要先进行memory的回收！

eg:
c++中要调用`out_params = color_tracker3(params,params)` 这样一个函数，而params 和out_params都是在c++程序中跟新和管理的。其中params 里有一个field “image”，每次传进去之前都需要重新对“image”进行赋值，那么此时就要注意将原先的“image”回收掉：

```
	mxArray *temp = mxGetField(out_params, 0, "im");
	mxDestroyArray(temp);

	mxSetField(out_params, 0, "im", MxArray(img));
```
上面程序的memory泄露主要是因为在c++只用了一个参数来进行参数的跟新，而且*color_tracker3函数的*out_params也将上一次的image给传出来了，而且matlab中`out_params = engGetVariable(ep, "out_params");` 是将参数复制一份，所以在进行原先的赋值操作时候将out_params传出来的”image“内存给泄露了.

综上：
如果想对matlab struct 中的field进行修改的时候首先加上防止memory leak
`	mxArray *temp = mxGetField(out_params, 0, "im");
	mxDestroyArray(temp);`
如果对于整个struct都不需要的话仅需对该struct进行`mxDestroyArray`，此函数是迭代进行的，会将里面的所有成员空间全部释放.


