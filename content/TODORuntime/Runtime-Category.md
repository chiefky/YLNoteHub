# Runtime-Category

## 1. 探究Category 本质

```objective-c

@interface YLNMyClass_runtime : NSObject

@property (nonatomic,copy) NSString *name;
- (void)printName;

@end

@interface YLNMyClass_runtime (MyAddition)

@property (nonatomic,copy) NSString *categoryName;

- (void)printName;
- (void)printCategoryName;

@end
	
```

```objective-c
@implementation YLNMyClass_runtime

- (void)printName
{
    NSLog(@"%@",@"YLNMyClass_runtime");
}

@end

@implementation YLNMyClass_runtime (MyAddition)

- (void)printName
{
    NSLog(@"%@",@"YLNMyClass_runtime+MyAddition");
}

- (void)printCategoryName {
    NSLog(@"%@_%@",@"YLNMyClass_runtime+MyAddition",@"MyAddition");
}

@end

```

`clang -rewrite-objc YLNMyClass_runtime.m`编译后得到YLNMyClass_runtime.cpp文件：

关键代码如下：

**Category结构体**

```cpp
struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};

// 构造函数
static struct _category_t _OBJC_$_CATEGORY_YLNMyClass_runtime_$_MyAddition1 __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"YLNMyClass_runtime",
	0, // &OBJC_CLASS_$_YLNMyClass_runtime,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_YLNMyClass_runtime_$_MyAddition1,
	0,
	0,
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_YLNMyClass_runtime_$_MyAddition1,
};

```

**元类实例变量列表、属性列表、方法列表：**

```cpp
static struct /*_ivar_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count;
	struct _ivar_t ivar_list[1];
} _OBJC_$_INSTANCE_VARIABLES_YLNMyClass_runtime __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_ivar_t),
	1,
	{{(unsigned long int *)&OBJC_IVAR_$_YLNMyClass_runtime$_name, "_name", "@\"NSString\"", 3, 8}}
};

static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[5];
} _OBJC_$_INSTANCE_METHODS_YLNMyClass_runtime __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	5,
	{{(struct objc_selector *)"printName", "v16@0:8", (void *)_I_YLNMyClass_runtime_printName},
	{(struct objc_selector *)"name", "@16@0:8", (void *)_I_YLNMyClass_runtime_name},
	{(struct objc_selector *)"setName:", "v24@0:8@16", (void *)_I_YLNMyClass_runtime_setName_},
	{(struct objc_selector *)"name", "@16@0:8", (void *)_I_YLNMyClass_runtime_name},
	{(struct objc_selector *)"setName:", "v24@0:8@16", (void *)_I_YLNMyClass_runtime_setName_}}
};

static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_YLNMyClass_runtime __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	1,
	{{"name","T@\"NSString\",C,N,V_name"}}
};

```

**分类1属性列表、方法列表：**

```cpp
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[2];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_YLNMyClass_runtime_$_MyAddition1 __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	2,
	{{(struct objc_selector *)"printName", "v16@0:8", (void *)_I_YLNMyClass_runtime_MyAddition1_printName},
	{(struct objc_selector *)"printCategoryName", "v16@0:8", (void *)_I_YLNMyClass_runtime_MyAddition1_printCategoryName}}
};

static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_YLNMyClass_runtime_$_MyAddition1 __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	1,
	{{"categoryName","T@\"NSString\",C,N"}}
};

```

**分类2属性列表、方法列表：**

```cpp
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_YLNMyClass_runtime_$_MyAddition2 __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"printName", "v16@0:8", (void *)_I_YLNMyClass_runtime_MyAddition2_printName}}
};

static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_YLNMyClass_runtime_$_MyAddition2 __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	1,
	{{"categoryName","T@\"NSString\",C,N"}}
};

```

**.cpp文件最后**

```cpp
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [2] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
	&_OBJC_$_CATEGORY_YLNMyClass_runtime_$_MyAddition1,
	&_OBJC_$_CATEGORY_YLNMyClass_runtime_$_MyAddition2,
};
```



综上：

1. 首先编译器生成了实例方法列表`_OBJC_$_CATEGORY_INSTANCE_METHODS_YLNMyClass_runtime_$_MyAddition1` 和属性列表`_OBJC_$_PROP_LIST_YLNMyClass_runtime_$_MyAddition1`，两者的命名都遵循了公共前缀+类名+category名字的命名方式。还有一个需要注意到的事实就是category的名字用来给各种列表以及后面的category结构体本身命名，而且有static来修饰，所以在同一个编译单元里我们的category名不能重复，否则会出现编译错误。
2. 其次，编译器生成了category本身`_OBJC_$_CATEGORY_YLNMyClass_runtime_$_MyAddition1`，并用前面生成的列表来初始化category本身。
3. 最后，编译器在**DATA段下的**objc_catlist section里保存了一个大小为1的category_t的数组`L_OBJC_LABEL_CATEGORY_$`（有多个category，会生成对应长度的数组），用于运行期category的加载。



## 2. Category如何加载

### 2.1 dyld 加载的流程

之前我们谈到过 Category（分类）是在运行时阶段动态加载的。而 Runtime（运行时） 加载的过程，离不开一个叫做 dyld 的动态链接器。

在 MacOS 和 iOS 上，动态链接加载器 dyld 用来加载所有的库和可执行文件。而加载Runtime（运行时） 的过程，就是在 dyld 加载的时候发生的。

dyld 的相关代码可在苹果开源网站上进行下载。 链接地址：[dyld 苹果开源代码](https://opensource.apple.com/tarballs/dyld/)

dyld 加载的流程大致是这样：

1. 配置环境变量；
2. 加载共享缓存；
3. 初始化主 APP；
4. 插入动态缓存库；
5. 链接主程序；
6. 链接插入的动态库；
7. 初始化主程序：OC, C++ 全局变量初始化；
8. 返回主程序入口函数。

本文中，我们只需要关心的是第 7 步，因为 Runtime（运行时）  是在这一步初始化的。加载 Category（分类）自然也是在这个过程中。

> 初始化主程序中，Runtime 初始化的调用栈如下：
>
> ```
> dyldbootstrap::start` --->  `dyld::_main` --->  `initializeMainExecutable` ---> `runInitializers` ---> `recursiveInitialization` ---> `doInitialization` ---> `doModInitFunctions` ---> `_objc_init
> ```

调用堆栈图：

<img src="./objc_init.png" alt="调用堆栈图" style="zoom:50%;" />

最后调用的 `_objc_init` 是 `libobjc` 库中的方法， 是 Runtime 的初始化过程，也是 Objective-C 的入口。

运行时相关的代码可在苹果开源网站上进行下载。 链接地址： [objc4 苹果开源代码](https://opensource.apple.com/tarballs/objc4/)

在 `_objc_init` 这一步中：`Runtime` 向 `dyld` 绑定了回调，当 `image` 加载到内存后，`dyld` 会通知 `Runtime` 进行处理，`Runtime` 接手后调用 `map_images` 做解析和处理，调用 `_read_images` 方法把 `Category（分类）` 的对象方法、协议、属性添加到类上，把 `Category（分类）` 的类方法、协议添加到类的 `metaclass` 上；接下来 `load_images` 中调用 `call_load_methods` 方法，遍历所有加载进来的 `Class`，按继承层级和编译顺序依次调用 `Class` 的 `load` 方法和其 `Category` 的 `load` 方法。

> 加载Category（分类）的调用栈如下：
>
> `_objc_init` ---> `map_images` ---> `map_images_nolock` ---> `_read_images（加载分类）` ---> `load_images`。

既然我们知道了 Category（分类）的加载发生在 `_read_images` 方法中，那么我们只需要关注`_read_images` 方法中关于分类加载的代码即可。

### 2.2 `_read_images()`方法内部实现

