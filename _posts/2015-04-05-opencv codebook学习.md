---
layout: post
title: opencv codebook学习
category: 科研
comments: true
---


该算法对图像的每一个像素位置建立一个码本(codebook),每一个码本有多个码元（ce）多通道的，每一个码元都有自己的lowbound和upbound。在学习时候，对于每一张图片中的每一个像素，进行相应的码本匹配，如果像素值在码本中某码元的bound范围之内，那么只需要更新该码元的bound即可。如果在对应的码本中没有匹配的码元，那么证明背景是动态的，需要在此像素点的码本中建立新的码元。因此，在背景学习的过程中，每个像素点可以对应多个码元，这样就可以学到复杂的动态背景。
1、学习过程
2、检测过程

```c++

maxMod[n]：用训练好的背景模型进行前景检测时用到，判断点是否小于max[n] + maxMod[n])。

minMod[n]：用训练好的背景模型进行前景检测时用到，判断点是否小于min[n] -minMod[n])。

cbBounds*：训练背景模型时用到，可以手动输入该参数，这个数主要是配合high[n]和low[n]来用的。

learnHigh[n]：背景学习过程中当一个新像素来时用来判断是否在已有的码元中，是阈值的上界部分。

learnLow[n]：背景学习过程中当一个新像素来时用来判断是否在已有的码元中，是阈值的下界部分。

max[n]： 背景学习过程中每个码元学习到的最大值，在前景分割时配合maxMod[n]用的。

min[n]： 背景学习过程中每个码元学习到的最小值，在前景分割时配合minMod[n]用的。

high[n]：背景学习过程中用来调整learnHigh[n]的，如果learnHigh[n]<high[n],则learnHigh[n]缓慢加1

low[n]： 背景学习过程中用来调整learnLow[n]的，如果learnLow[n]>Low[n],则learnLow[缓慢减1
```

***数据结构：***
```c++

typedef struct ce {
	uchar   learnHigh[CHANNELS];    // High side threshold for learning
	uchar   learnLow[CHANNELS];     // Low side threshold for learning
	uchar   max[CHANNELS];          // 当前码元在学习中遇到的最大像素值
	uchar   min[CHANNELS];          // 当前码元在学习中遇到的最小像素值
	int     t_last_update;          // This is book keeping to allow us to kill stale entries
	int     stale;                  // max negative run (biggest period of inactivity)，记录该码元未被访问的最长时间
} code_element;                     // 码元的数据结构
	
typedef struct code_book {
	code_element    **cb;
	int             numEntries;     // 此码本中码元的数目
	int             t;              // 此码本现在的时间,一帧为一个时间单位
} codeBook;                         //每个像素点位置有一个码本

```

***成员初始化：***

```c++
Codebook::Codebook()
{
	yuvImage = NULL;//原始图像
	ImaskCodeBook = NULL;
	ImaskCodeBookCC = NULL;
	cB = NULL;
	pColor = NULL; //YUV pointer
	imageLen = 0;//像素大小
	nChannels = CHANNELS;

	approxLevel = 0;//
	closeIter = 2;
	maxIterInLearn = 100;
	tolerance = 30;//进行检测时候的上下波动值
	perimScale = 7.5;//连通域删选的周长倍数阈值
	isFirstRun = true;
	iterInLearn = 1;
	isNewFrame = false;

	isShowMask = false;
}
```


***主程序***

```c++

int main()
{
	Mat img;
	VideoCapture *cap;

	cap = new VideoCapture("...\\temp_Short.avi");

	Codebook cb;
	int key = 0;

	int iter = 0;
	while (1)
	{
		*cap >> img;
		if (img.empty())
			break;
		key = waitKey(1);
		if (key == 's')   //暂停
		{
			while (1)
			{
				if (waitKey(1) == 'r')
					break;
			}
		}
		if (iter < 100)//此为训练张数
		{
			cb.learn(img);//进行训练
		}
		else
		{
			imshow("mask", cb.process(img));
		}
		iter++;
		imshow("show", img);

		waitKey(32);
	}

	delete cap;
}

```

***初始化函数***：rawframe即为当前的frame
``` c++
int Codebook::initial()
{
	if (yuvImage)
		cvReleaseImage(&yuvImage);
	if (ImaskCodeBook)
		cvReleaseImage(&ImaskCodeBook);
	delete[] cB;

	yuvImage = cvCreateImage(cvGetSize(rawImage), 8, 3);
	ImaskCodeBook = cvCreateImage(cvGetSize(rawImage), IPL_DEPTH_8U, 1);
	ImaskCodeBookCC = cvCreateImage(cvGetSize(rawImage), IPL_DEPTH_8U, 1);

	cvSet(ImaskCodeBook, cvScalar(255));  // 设置单通道数组所有元素为255,即初始化为白色图像
	imageLen = rawImage->width * rawImage->height;
	cB = new codeBook[imageLen];  // 得到与图像像素数目长度一样的一组码本,以便对每个像素位置进行处理

	for (int i = 0; i<imageLen; i++)   // 初始化每个码元数目为0
		cB[i].numEntries = 0;
	for (int i = 0; i<nChannels; i++)
	{
		cbBounds[i] = 10;   // 像素值邻域,仅在更新学习过程中的上下阈值learnHigh和learnLow时用到。high = *(p + n) + *(cbBounds + n),low = low[n] = *(p + n) - *(cbBounds + n),即当前像素点像素值的正负10之内
		minMod[i] = tolerance;   // 用于背景差分函数中
		maxMod[i] = tolerance;   // 调整其值以达到最好的分割
	}

	show = cvCreateImage(cvGetSize(rawImage), IPL_DEPTH_8U, 3);

	return 1;
}
``` 

***学习函数：***


```
void Codebook::learn(Mat frame)
{
	if (isFirstRun)
	{
		rawImage = cvCreateImage(cvSize(frame.cols, frame.rows), IPL_DEPTH_8U, 3);
		*rawImage = frame;
		initial();//初始化将所有成员变量进行初始
		isFirstRun = false;
	}
	if (iterInLearn < maxIterInLearn)  //进行背景学习
	{
		*rawImage = frame;
		cvCvtColor(rawImage, yuvImage, CV_BGR2YCrCb);
		pColor = (uchar *)(yuvImage->imageData);
		for (int c = 0; c<imageLen; c++)
		{
			cvupdateCodeBook(pColor, cB[c], cbBounds, nChannels);//对每一个码本进行训练
			pColor += 3;
		}
	}
	else if (iterInLearn == maxIterInLearn)//训练完成，进行codebook的精简，删除长时间未访问的码元
	{
		for (int c = 0; c<imageLen; c++)//需要对每一个码本进行操作
			cvclearStaleEntries(cB[c]);
	}
	iterInLearn++;
}

```



***学习过程cvupdateCodeBook（）***

如果当前像素值（x = （*p））在codebook中找到一个码元cb[i],使得每一通道x的值都在cb[i]的[learnLow[n] , learnHigh[n]]区间，那么就不用增加码元，需检查当前的像素值是否为match的码元的最大值或者最小值，并进行更新，此外还要对learnLow[n] , learnHigh[n]也进行缓慢的调整：

检查是否有匹配码元：

```c++
int matchChannel;
	int i;
	for (i = 0; i<c.numEntries; i++)
	{
		matchChannel = 0;
		for (n = 0; n<numChannels; n++)
		{
			if ((c.cb[i]->learnLow[n] <= *(p + n)) && (*(p + n) <= c.cb[i]->learnHigh[n])) //Found an entry for this channel
			{
				matchChannel++;
			}
		}
		if (matchChannel == numChannels)        // If an entry was found over all channels
		{
			c.cb[i]->t_last_update = c.t;
			// adjust this codeword for the first channel
			for (n = 0; n<numChannels; n++)
			{
				if (c.cb[i]->max[n] < *(p + n))
					c.cb[i]->max[n] = *(p + n);
				else if (c.cb[i]->min[n] > *(p + n))
					c.cb[i]->min[n] = *(p + n);
			}
			break;
		}
	}
```

如果找到匹配码元，即每个通道的值都在范围内

```c++
		if (matchChannel == numChannels)        // If an entry was found over all channels
		{
			c.cb[i]->t_last_update = c.t;
			// adjust this codeword for the first channel
			for (n = 0; n<numChannels; n++)
			{
				if (c.cb[i]->max[n] < *(p + n))
					c.cb[i]->max[n] = *(p + n);
				else if (c.cb[i]->min[n] > *(p + n))
					c.cb[i]->min[n] = *(p + n);
			}
			break;
		}
```

缓慢调整的规则：比较 x像素值cvbounds领域与learnLow[n] , learnHigh[n]的大小。如果learnLow[n] > x - cvbounds，那么 learnLow- -；

```c++
	// SLOWLY ADJUST LEARNING BOUNDS
	for (n = 0; n<numChannels; n++)
		//如果像素通道数据在高低阀值范围内,但在码元阀值之外,则缓慢调整此码元学习界限
		//也就是像素命中码元了（若没有命中就会新建，就learnHigh[n] == high[n]）
		//要命中，而且码元的阈值不是特别大（与像素值的差不足cvBounds）时就扩大码元学习限
	{
		if (c.cb[i]->learnHigh[n] < high[n])
			c.cb[i]->learnHigh[n] += 1;
		if (c.cb[i]->learnLow[n] > low[n])
			c.cb[i]->learnLow[n] -= 1;
	}
```

如果当前没有匹配的码元，那么需要给此像素的码本增加新的码元：

```c++
		learnHigh[CHANNELS] = *(p + n) + *(cbBounds + n)；//当前像素值+cbBounds 
		learnLow[CHANNELS] = *(p + n) - *(cbBounds + n);//当前像素值-cbBounds     
		max[CHANNELS]  =     *(p + n)  
		min[CHANNELS]  =     *(p + n) 
		t_last_update  =     c.t                        //当前时间
```
然后需要进行码元的回收判断，当一个码元在训练的时候长期没有被访问到（stale为最长不访问的时间），那么证明该码元已经不适用，那么我们可以把它删掉，以减少内存消耗。更新最长未访问时间：

```c++
	for (int s = 0; s<c.numEntries; s++)
	{
		// This garbage is to track which codebook entries are going stale
		int negRun = c.t - c.cb[s]->t_last_update;

		if (c.cb[s]->stale < negRun)
			c.cb[s]->stale = negRun;
	}
```
ps：《学习opencv》给的代码中cvupdateCodeBook()有问题：原来给的是unsigned int，应该改为int。

```c++
'unsigned' int high[3], low[3];//应该是int 不是unsigned int，不然如果有0、0、0那么low不会小于0！！
for (n = 0; n<numChannels; n++)
{
	high[n] = *(p + n) + *(cbBounds + n);
	if (high[n] > 255) high[n] = 255;
	low[n] = *(p + n) - *(cbBounds + n);
	if (low[n] < 0) low[n] = 0;
}
```



***删除长期未访问的码元***
此操作只在背景学习结束后马上执行，且仅执行一次。
其中认为outdate的时间阈值为学习总时间的一半： 'staleThresh = c.t >> 1'

```c++
int Codebook::cvclearStaleEntries(codeBook &c)//返回删除的码元个数
{
	int staleThresh = c.t >> 1;
	int *keep = new int[c.numEntries];//标记码元是否有效
	int keepCnt = 0;//记录有效码元个数
	//筛选未访问时间没有超过staleThresh的码元
	for (int i = 0; i<c.numEntries; i++)
	{
		if (c.cb[i]->stale > staleThresh)
			keep[i] = 0;
		else
		{
			keep[i] = 1;
			keepCnt += 1;
		}
	}
	
	c.t = 0;
	
	//将未过期的码元建立成新的码本
	code_element **foo = new code_element*[keepCnt];
	
	int k = 0;
	for (int ii = 0; ii<c.numEntries; ii++)
	{
		if (keep[ii])
		{
			foo[k] = c.cb[ii];
			foo[k]->stale = 0;       //We have to refresh these entries for next clearStale
			foo[k]->t_last_update = 0;
			k++;
		}
	}
	delete[] keep;
	delete[] c.cb;
	c.cb = foo;//将新码本代替原来的码本
	int numCleared = c.numEntries - keepCnt;
	c.numEntries = keepCnt;
	return(numCleared);
}
```



***前背景检测***
和学习时候对码元是否已经存在类似，规则如下：
对于检测图片，扫描每个像素，当前像素值x在当前码本中找到一个码元，使得x属于当前码元的[min[n] - minMod[n],max[n] + maxMod[n]],那么认为是背景，否则为前景。

```c++
for (int c = 0; c<imageLen; c++)
	{
		maskPixelCodeBook = cvbackgroundDiff(pColor, cB[c], nChannels, minMod, maxMod);
		*pMask++ = maskPixelCodeBook;
		pColor += 3;
	}


uchar Codebook::cvbackgroundDiff(uchar *p, codeBook &c, int numChannels, int *minMod, int *maxMod)
{
	int matchChannel;
	//SEE IF THIS FITS AN EXISTING CODEWORD
	int i;
	for (i = 0; i<c.numEntries; i++)
	{
		matchChannel = 0;
		for (int n = 0; n<numChannels; n++)
		{
			if ((c.cb[i]->min[n] - minMod[n] <= *(p + n)) && (*(p + n) <= c.cb[i]->max[n] + maxMod[n]))
				matchChannel++; //Found an entry for this channel
			else
				break;
		}
		if (matchChannel == numChannels)
			break; //Found an entry that matched all channels
	}
	if (i == c.numEntries)
		// p像素各通道值满足码本中其中一个码元,则返回白色
		return(255);

	return(0);
}
```

**在检测的过程中我们也可以持续的更新codebook，即延时nGapFrame帧然后用其进行训练，并且在训练部分后进行codebook冗余的删除，这样对于运动的检测会更加准确。**

```c++
//动态更新背景,延时nGapFrame帧
if (StartUpdate)
{
	IplImage *temp = cvCreateImage(cvSize(frame.cols, frame.rows), IPL_DEPTH_8U, 3);;
	*temp = gapFrameSet.front();
	cvCvtColor(temp, temp, CV_BGR2YCrCb);
	pColor = (uchar *)(temp->imageData);
	for (int c = 0; c < imageLen; c++)
	{
		cvupdateCodeBook(pColor, cB[c], cbBounds, nChannels);//对每一个码本进行训练
		pColor += 3;
	}
	gapFrameSet.erase(gapFrameSet.begin());
}

//每经过100帧动态进行码本的删除冗余
if (iterInTest % nGapFrame == 0)
{
	for (int c = 0; c < imageLen; c++)
		cvclearStaleEntries(cB[c]);
}
gapFrameSet.push_back(frame.clone());

//间隔nGapFrame帧后开始进行背景的动态更新
if (iterInTest % nGapFrame == 0)
{
	if (StartUpdate == false)
		StartUpdate = true;
	else
		StartUpdate = false;
}
	
```



***提取运动物体***
通过寻找连通域并且进行形态学的处理来提取运动物体：

```c++
Mat Codebook::cvconnectedComponents(IplImage *mask, int poly1_hull0, float perimScale,Mat frame)
{
	static CvMemStorage*    mem_storage = NULL;
	static CvSeq*           contours = NULL;

	cvMorphologyEx(mask, mask, NULL, NULL, CV_MOP_OPEN, closeIter);
	// 开运算先腐蚀在膨胀,腐蚀可以清除噪点,膨胀可以修复裂缝
	cvMorphologyEx(mask, mask, NULL, NULL, CV_MOP_CLOSE, closeIter);
	// 闭运算先膨胀后腐蚀,之所以在开运算之后,因为噪点膨胀后再腐蚀,是不可能去除的

	if (mem_storage == NULL)
		mem_storage = cvCreateMemStorage(0);
	else cvClearMemStorage(mem_storage);

	CvContourScanner scanner = cvStartFindContours(mask, mem_storage,
		sizeof(CvContour), CV_RETR_EXTERNAL);

	CvSeq* c;
	int numCont = 0;



	while ((c = cvFindNextContour(scanner)) != NULL)
	{
		double len = cvContourPerimeter(c);
		// 计算轮廓周长
		double q = (mask->height + mask->width) / perimScale;

		if (len < q)
			cvSubstituteContour(scanner, NULL);//由于周长过小，删除当前的这一连通域
		else
		{
			CvSeq* c_new;
			if (poly1_hull0)
				c_new = cvApproxPoly(c, sizeof(CvContour), mem_storage,
				CV_POLY_APPROX_DP, approxLevel);//使用多边形逼近轮廓
			else
				c_new = cvConvexHull2(c, mem_storage, CV_CLOCKWISE, 1);

			cvSubstituteContour(scanner, c_new);
			numCont++;
		}
	}


	contours = cvEndFindContours(&scanner);

	cvZero(mask);

	for (c = contours; c != NULL; c = c->h_next)
		cvDrawContours(mask, c, CV_CVX_WHITE, CV_CVX_BLACK, -1, CV_FILLED, 8);
}
```

