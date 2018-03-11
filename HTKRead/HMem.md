# HMem
### HTK堆内存
htk堆内存管理方式分为三种：

- MHEAP 存放固定大小对象，new和free次序无限制
- MSTACK 存放的对象大小可变，最后分配的空间先释放
- CHEAP 存放的对象大小可变，new和free次序无限制，不常用

### 内存数据结构
~~~c
	typedef struct {				/*  MHEAP         MSTAK */
		char *name;				/* 			名称         */
		HeapType type;         /* 			类型         */
		float growf;				/* 块元素个数增长因子 块字节数增长因子    */
		size_t elemSize;			/* 元素大小          1   */
		size_t minElem;			/* 块初始元素个数  块初始字节数 */
		size_t maxElem;        /* 块最大元素个数  块最大字节数 */
		size_t curElem;			/* 块当前元素个数  块当前字节数 */
		size_t totUsed;        /* 已使用元素个数  已使用字节数 */
		size_t totAlloc;       /* 总元素个数      总字节数 */
		BlockP heap;           /*         块链表       */
		Boolean proteckStk;    /* 用于MSTACK实现后进先出  */
	} MemHeap;
	
	typedef struct {
		size_t numFree;			/* 空闲元素个数    空闲字节数*/
		size_t firstFree;      /* 第一个空闲元素索引 栈顶索引*/
		size_t numElem;        /* 块元素个数      块字节数*/
		ByteP used;            /* 指向元素状态表   未使用*/
		Ptr data;              /*        指向数据区     */
		BlockP next;
	} Block;
~~~
结构示意图如下：	
![MemHeap icon](http://img.blog.csdn.net/20140901172805404?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnlsb3ZlMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
![MemHeap icon2](http://img.blog.csdn.net/20140901173002763?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnlsb3ZlMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

内存堆通过链表进行跟踪，定义如下：

~~~c
	typedef struct _MemHeapRec {
		MemHeap *heap;
		struct _MemHeapRec *next;
	} MemHeapRec;
	
	static MemHeapRec *heapList = NULL;
~~~

### 内存操作函数
~~~c
	/* 创建具有给定属性的堆内存，并放入堆链表heapList中，对于MSTACK，elemSize值必须为1 */
	void CreateHeap(MemHeap *x, char *name, HeapType type, size_t elemSize, float growf, size_t numElem, size_t maxElem);
	
	/* 释放x中所有元素，CHEAP不能释放，对MSTACK，第一个Block内存不释放，但标记为未使用，原因未知？ */
	void ResetHeap(MemHeap *x);
	
	/* 释放x中所有元素和x内存堆，并从heapList中删除x */
	void DeleteHeap(MemHeap *x);
	
	/* 返回堆x中大小为size的新元素指针 */
	Ptr New(MemHeap *x, size_t size);
	
	/* 从内存堆x中释放元素p，只标记为0，但如果释放完发现元素p所在块未使用，则释放该块 */
	void Dispose(MemHeap *x, void *p);
	
	/* 获取块中一个空闲元素指针，如果是MHeap，置其在使用状态表中的值为1；如果是MStack，返回大小为elemSize字节数的区域指针 */
	static void *GetElem(BlockP p, size_t elemSize, HeapType type);
~~~
### vector和Matric内存
数据结构

~~~c
	/* 向量 [1,...,size]，第0个元素存放size */
	typedef short * ShortVec;
	typedef int * IntVec;
	typedef float * Vector;
	typedef double ** DVector;
	
	/* 矩阵 [1,...,nrows][1,...,ncols] */
	typedef float ** Matrix;
	typedef double ** DMatrix;
	/* [1,...,nrows][1,...,i] */
	typedef Matrix TriMat;
~~~
Matrix在内存中示意图如下：
![Matix icon](http://pic002.cnblogs.com/images/2011/285259/2011032510454697.jpg)
所占内存：`vectorElemSize(C)*R+sizeof(Vector)*(R+1)`

TriMat示意图如下：
![TriMat icon](http://pic002.cnblogs.com/images/2011/285259/2011032510481284.jpg)
所占内存：`R*sizeof(float)+C*sizeof(float)+sizeof(float)*C(C+1)/2`

操作函数

~~~c
	size_t VectorElemSize(int size);
	Vector CreateVector(MemHeap *x, int size);
	int VectorSize(Vector v);
	void FreeVector(MemHeap *x, Vector v);
	
	size_t MatrixElemSize(int nrows, int ncols);
	Matrix CreateMatrix(MemHeap *x, int nrows, int cols);
	int NumRows(Matrix m);
	int NumCols(Matrix m);
	void FreeMatrix(MemHeap *x, Matrix m);
~~~

### 字符串内存
~~~c
	char *NewString(MemHeap *x, int size);
	char *CopyString(MemHeap *x, char *s);
~~~