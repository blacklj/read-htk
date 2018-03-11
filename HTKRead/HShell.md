# HShell
### 出错处理
	/* 退出，可以配置是否打印配置参数 */
	void Exit(int exitcode);
	
	/* 出错，错误信息格式化输出到stderr */
	void HError(int errorcode, char *message, ...);

### 初始化
###### 脚本中存放命令行参数
`ReturnStatus SetScriptFile(char *fn)`初始化`scriptcount`的值

###### HShell初始化
`ReturnStatus InitShell(int argc, char *argv, char *ver, char *sccs);`处理过程如下：

1. 检查环境变量中是否设置默认参数文件`HCONFIG`，如果设置，读取其中的参数信息
2. 将命令行参数存储到全局静态变量`saveCommandLine`中，需要时通过`char *RetrieveCommandLine(void)`函数获取
3. 判断主机子节序是否为小端`static Boolean IsVAXOrder()`
4. 遍历命令行参数，处理`-A -C -S -V -D`选项
	
	`-A`表示输出命令行参数到标准输出
	
	`-C`后面是配置文件，通过`ReadConfigFile(char *name)`读取其中的参数
	
	`-S`后面的文件存放的是脚本命令script，通过`SetScriptFile(char *fn)`读取命令的个数
		
	`-V`打印各个工具的版本信息

	`-D`打印版本和配置参数信息

5. 其余参数存储在arglist中
6. 打印配置参数信
7. 读取并初始化`HSHELL`模块的配置参数
	
###### 注册版本和SCCS信息
`void Register(char *ver, char *sccs)`，所有版本信息通过一个带头尾指针的链表管理

###### 注册扩展文件名
`char *RegisterExtFileName(char *s)`，扩展文件名定义格式有三种类型：

	file[s,e];
	logfile=actfile;
	logicName=physicsName[s,e];
	
表示处理从physicsName文件中的s位置开始到e位置结束的内容，内部数据结构定义如下：

	typedef struct {
		char logfile[1024];		/* 文件逻辑名 */
		char actfile[1024];		/* 文件物理名 */
		long stindex;			/* 文件的起始位置 */
		long enindex;			/* 文件的结束位置 */
	}ExtFile;
		
这里使用了数组充当循环缓冲区，存储`ExtFile`类型对象，并定义了两个变量`static int extFileNext`和`static int extFileUsed`分别计数对象的位置和个数。

`Boolean GetFileNameExt(char *logfn, char *actfn, long *st, long *en)`函数用于从数组中获取逻辑名为logfn的扩展文件信息。

值得注意的是，该函数是通过比较logfn和extFile中logfile的地址是否相同来获取文件信息的的。而由于扩展文件数组中可能存在多个逻辑名相同的扩展文件，因此当地址匹配失败时，重新通过字符串比较逻辑名，此时如果有多个逻辑名相同的对象时会报错。

###### 打印版本信息

	Boolean InfoPrinted(void); /* 模块初始化完后即刻调用 */

###### 打印命令行选项

	void PrintStdOpts(char *opts); /* 详见htkbook */
	
### 命令行参数处理
###### 数据结构定义
	
	typedef enum {		/* 命令行参数类型 */
		SWITCHARG,		/* 以'-'开头的字符串 */
		STRINGARG,		
		INTARG,
		FLOATARG,
		NOARG			/* 没有参数时返回的类型 */
	} ArgKind;
	
###### 函数定义

	int NumArgs(void); 	/* 返回未处理的命令行参数个数，return argc+scriptcount-nextarg */
	
	ArgKind NextArg(void); /* 返回下一个参数的类型 */
	
	/* 获取各种类型参数 */
	char *GetStrArg();
	char *GetSwtArg();
	int GetIntArg();
	long GetLongArg();
	float GetFltArg();
	
	/* 获取并检查参数类型和取值范围，swtname是选项名，如-A */
	int GetChkedInt(int min, int max, char *swtname);
	long GetChkedLong(long min, long max, char *swtname);
	float GetChkedFlt(float min, float max, char *swtname);
	
	Boolean GetIntEnvVar(char *envVar, int *value);
	

### 文件配置参数
###### 数据结构定义

	typedef struct {
		char *user;			/* 参数所属模块 */
		char *name;			/* 参数名 */
		ConfKind kind;		/* 参数类型 */
		ConfVal val;		/* 参数值  */
		Boolean seen;		/* ? */
	}
参数值定义如下，一个联合体支持字符串类型、整形、浮点型和bool型。

	typedef union {
		char *s;
		int i;
		double f;
		Boolean b;
	} ConfVal;
参数通过一个叫做`ConfigEntry`类型的串联起来，类似一个参数链表，定义如下：

	typedef struct _ConfigEntry {
		ConfigParam param;
		struct _ConfigEntry *next;
	} ConfigEntry;
	
###### 读取配置文件
`ReadConfigFile(char *fname)`函数实现读取配置文件，产生配置参数链表confList,其实现过程简介如下：

1. 判断达到最大`#include`数(默认为15)
2. 初始化Source对象，指向配置文件，函数为`InitSource(char *fname, Source*src, IOFilter filter)`
3. 处理注释和`#include`文件，`ParseComment(Source *src, char *fname)`
4. 读取参数名称`ReadConfName(Source *src, char *s)`
5. 读取参数值`ReadString(Source *src, char *s)`
6. 判断参数是否在参数链表中，如果不在则加入到链表中
7. 根据参数值，设置参数的类型
8. 处理注释和`#include`	// ?
9. 读取参数名称，调到第5步，直到所有参数读取结束

###### 打印配置参数

	void PrintConfig(void);
	
###### 获取指定模块参数

	int GetConfig(char *user, Boolean incGlob, ConfParam **list, int max);
模块初始化时调用`GetConfig`从confList中取得本模块的配置参数。

###### 配置参数操作函数

	Boolean HasConfParm(ConfParam **list, int size, char *name);
	Boolean GetConfStr(ConfParam **list, int size, char *name, char *str);
	Boolean GetConfBool(ConfParam **list, int size, char *name, Boolean *b);
	Boolean GetConfInt(ConfParam **list, int size, char *name, int *ival);
	Boolean GetConfFlt(ConfParam **list, int size, char *name, double *fval);
	
这些函数是对`static int FindConfParam(ConfParam **list, int size, char *name, ConfKind kind)`函数的封装。该函数遍历list，返回查找到name和kind满足条件的参数的索引。

### 输入处理
###### 打开关闭文件

	FILE *FOpen(char *fname, IOFilter filter, Boolean *isPipe);
	void FClose(FILE *f, Boolean isPipe);
	
###### Source定义
用来指代一个文件，方便进行文件读写操作，定义如下：
	
	typedef struct {
		char name[256];			/* 文件名 */
		FILE *f;				/* 文件操作符 */
		Boolean isPipe;			/* 是否管道文件 */
		Boolean pbValid;		/* 回放字符非空时为true */
		Boolean wasQuoted;		/* 是否是引用字符串 */
		Boolean wasNewline;		/* 是否换行符 */
		int putback;			/* 回放字符 */
		int chcount;			/* 起始位置起的字符数 */
	}Source;
支持的操作如下：

	/* 初始化src指向fname */
	ReturnStatus InitSource(char *fname, Source *src, IOFilter filter);
	
	/* 关闭src */
	void CloseSource(Source *src);
	
	/* 关联文件到src*/
	void AttachSource(FILE *f, Source *src);
	
	/* 描述src当前位置 */
	char *SrcPosition(Source src, char *s);
	
	/* 读取下一个字符 */
	int GetCh(Source src);
	
	/* 将指定字符放回src */
	void UnGetCh(int c, Source *src);
	
	/* 读取下一个字符串，存储在s中，字符串长度限制256个 */
	Boolean ReadString(Source *src, char *s);
	Boolean ReadStringWithLen(Source *src, char *s, int buflen);
	
	/* 读取原始字符串，忽略转移字符 */
	Boolean ReadRawString(Source *src, char *s);
	
	/* 跳过一行 */
	Boolean SkipLine(Source *src);
	
	/* 读取一行 */
	Boolean ReadLine(Source *src, char *s);
	
	/* 读取到指定的字符串 */
	void ReadUntilLine(Source *src, char *s);
	
	/* 读取n个int/float/short数据 */
	Boolean ReadInt(Source *src, short *s, int n, Boolean binary);
	Boolean ReadShort(Source *src, short *s, int n, Boolean binary);
	Boolean ReadFloat(Source *src, short *s, int n, Boolean binary);
	
### 输出处理
封装fwrite函数，如下：

	void WriteShort(FILE *f, short *s, int n, Boolean binary);
	void WriteInt(FILE *f, int *i, int n, Boolean binary);
	void WriteFloat(FILE *f, float *x, int n, Boolean binary);

### 文件名处理

	/* 获取完整文件名、不带扩展名的文件名、路径和扩展名 */
	char *NameOf(char *fn, char *s);
	char *BaseOf(char *fn, char *s);
	char *PathOf(char *fn, char *s);
	char *ExtOf(char *fn, char *s);
	
	/* 创建同名文件名 */
	char *MakeFN(char *fn, char *path, char *ext, char *s);
	
	/* 创建类似'prefNNNsuf'的文件名 */
	char *CounterFN(char *prefix, char *suffix, int count, int width, char *s);
	
	/* 以文件名替换配置参数中'$' */
	void SubstFName(char *fname, char *s);


### 模式匹配

	/* 支持?和*两种匹配方式 */
	Boolean RMatch(char *s, char *p, int slen, int minplen, int numstars);	
	Boolean DoMatch(char *s, char *p);
	
	/* 匹配%的存放在spkr中 */
	Boolean MaskMatch(char *mask, char *spkr, char *str);