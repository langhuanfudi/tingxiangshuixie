# 右值引用与移动语义

## 为何需要移动语义

假设我们要做这样一个复制过程

```c++
vector<string> vstr;
// 构建一个包含20000个string的vector, 每个string有1000个字符
vector<string> vstr_copy1(vstr); //vstr_copy1是vstr的复制
```

在vector类和string类中都会定义一个拷贝构造函数，其中使用了某个版本的new函数。为了初始化vstr_copy1，vector<string>的拷贝构造函数会用new来给这20000个string对象分配内存。而每个string对象又会用string中的拷贝构造函数来为1000个字符分配内存。总共会将20000000个字符从vstr控制的内存中拷贝到vstr_copy1控制的内存中。但是这有时候是不妥当的，比如：

```c++
vector<string> allcaps(const vector<string> &vs){
    vector<string> temp;
    //把vs中所有字符的大写存入temp
    return temp;
}

vector<string> vstr;
// 构建一个包含20000个string的vector, 每个string有1000个字符
vector<string> vstr_copy1(vstr);          //#1
vector<string> vstr_copy2(allcaps(vstr));	//#2
```

对语句#2来说，allcaps(vstr)创建了对象temp，管理了20000000个字符。vector类和string类的拷贝构造函数需要copy这200000000个字符，然后删除allcaps()返回的临时对象，做了大量的无用功。比较好的方法是，不需要copy这20000000个字符，而是直接将其所有权转让给vstr_copy2。这类似于在计算机中移动文件，文件实际的位置没变化，只是修改了记录。这被称为移动语义(move semantics)。

> 好比你家住在上海，现在你家位置完全没变，但把上海直接改名叫北京，然后你家就住在北京了

要实现移动语义需要通过某种方式让编译器知道，什么时候要copy，什么时候不要。右值引用就是干这个的，对于常规的拷贝构造函数，用const左值引用来作为参数，将这个引用关联到左值实参，如语句#1中的vstr。另外写一个移动构造函数，用右值引用作为参数，关联到右值实参，如语句#2中的allcaps(vstr)。拷贝构造函数可以执行深复制，移动构造函数只修改记录。移动构造函数要把所有权转移给新对象，可能修改其实参，意味着右值引用的参数不是const的。

## 一个移动示例

```c++
class Useless {
private:
    int n;         // number of elements
    char *pc;      // pointer to data
    static int ct; // number of objects
    void ShowObject() const;

public:
    Useless();
    explicit Useless(int k);
    Useless(int k, char ch);
    Useless(const Useless &f); // regular copy constructor
    Useless(Useless &&f);      // move constructor
    ~Useless();
    Useless operator+(const Useless &f) const;
    void ShowData() const;
};

Useless::Useless(const Useless &f) : n(f.n) {
    ++ct;
    cout << "copy const called; number of objects: " << ct << endl;
    pc = new char[n];
    for (int i = 0; i < n; i++) pc[i] = f.pc[i];
    ShowObject();
}

Useless::Useless(Useless &&f) {
    ++ct;
    cout << "move constructor called; number of objects: " << ct << endl;
    pc = f.pc; //steal address
    f.pc = nullptr;
    f.n = 0;
    ShowObject();
}
```

这里只列举了普通的拷贝构造函数和移动构造函数。拷贝构造函数执行了深复制，而移动构造函数直接将pc指向现有的数据来获取所有权。pc和f.pc不能指向同一个地址，因为析构时不能delete两次，所以要将原来的f.pc设为nullptr。这里修改了f，所以参数声明中不加const。



# iterators中的traits编程技法

## iterator的相应类别

在算法中运用iterator时很可能会用到它的相应类别(associated type)，也就是iterator所指的东西的类别。假设算法中要声明一个变量，以“iterator所指对象的类别”为类别，该怎么办？

可以利用function template的参数推导机制。

```c++
template<class I, class T>
void func_impl(I iter, T t) {
    T tmp; //T就是iter所指对象的类别

    // ... 这里做func()中的工作
    tmp = t;
    cout << tmp << endl; //输出5
}

template<class I>
inline void func(I iter) {
    //在这个函数中要声明一个变量，以iter所指对象的类别为类别
    func_impl(iter, *iter); //将func的工作全部移往func_impl
}

int main() {
    int i = 5;
    func(&i);
}
```

但是，iterator的相应类别不只“iter所指东西的类别”这一种，常见的有5种。并非任何情况下任何一种都可以用function template的参数推导机制来取得。

## 声明内嵌类别

template的参数推导机制只能推导参数，无法推导函数的返回值类别。

通过声明内嵌类别是个好主意！像这样：

```c++
//声明一个iterator
template<class T>
struct MyIter {
    typedef T value_type; //声明内嵌类别
    T *ptr;
    MyIter(T *p = 0) : ptr(p) {}
    T &operator*() const { return *ptr; }
};

template<class I>
typename I::value_type func(I iter) { //函数返回值是typename I::value_type
    return *iter;
}

int main() {
    MyIter<int> iter(new int(8));
    cout << func(iter) << endl; //输出8
}
```

但是不是所有iterator都是class type，原生指针就不是。class type无法为原生指针定义内嵌类别。

## 模板偏特化(template partial specialization)

模板偏特化的意义：如果class template有多个template参数，针对(任何)template参数更进一步的条件限制所设计出来的一个特化版本。

```c++
template<typename T>
class C{};

template<typename T> //仅适用于T为原生指针的情况的特化版本
class C<T*>{};
```

可以用下面的class template来萃取iterator的traits，value_type是traits之一

```c++
//如果I有自己的value_type，那萃取得到的value_type就是I::value_type
template<class I>
struct iterator_traits {
    typedef typename I::value_type value_type;
};

//针对iterator是原生指针的偏特化版本
template<class T>
struct iterator_traits<T*> {
    typedef T value_type;
};

//针对iterator是pointer-to-const时, 萃取的类别应该是T而非const T
template<class T>
struct iterator_traits<const T*>{
    typedef T value_type;
};
```



# Guide to predefined macros in C++ compilers (gcc, clang, msvc etc.)

When writing portable C++ code you need to write conditional code that depends on compiler used or the OS for which the code is written.

Here’s a typical case:

```
#if defined (_MSC_VER)
// code specific to Visual Studio compiler
#endif
```

To perform those checks you need to check pre-processor macros that various compilers set.

It can either be binary is defined vs. is not defined check (e.g. `__APPLE__`) or checking a value of the macro (e.g. `_MSC_VER` defines version of Visual Studio compiler).

This document describes macros set by various compilers.

Other documentations:

- predefined macros in [Visual Studio](https://msdn.microsoft.com/en-us/library/b0084kay.aspx)
- https://sourceforge.net/p/predef/wiki/Compilers/
- to list clang’s pre-defined macros: `clang -x c /dev/null -dM -E`
- to list gcc’s pre-defined macros: `gcc -x c /dev/null -dM -E` (not that on mac gcc is actually clang that ships with XCode)
- the `-x c /dev/null -dM -E` also works for mingw (which is based on gcc)
- listing predefined macros [for other compilers](http://nadeausoftware.com/articles/2011/12/c_c_tip_how_list_compiler_predefined_macros)

## Checking for OS (platform)

To check for which OS the code is compiled:

```
Linux and Linux-derived           __linux__
Android                           __ANDROID__ (implies __linux__)
Linux (non-Android)               __linux__ && !__ANDROID__
Darwin (Mac OS X and iOS)         __APPLE__
Akaros (http://akaros.org)        __ros__
Windows                           _WIN32
Windows 64 bit                    _WIN64 (implies _WIN32)
NaCL                              __native_client__
AsmJS                             __asmjs__
Fuschia                           __Fuchsia__
```

## Checking the compiler:

To check which compiler is used:

```
Visual Studio       _MSC_VER
gcc                 __GNUC__
clang               __clang__
emscripten          __EMSCRIPTEN__ (for asm.js and webassembly)
MinGW 32            __MINGW32__
MinGW-w64 32bit     __MINGW32__
MinGW-w64 64bit     __MINGW64__
```

## Checking compiler version

###  gcc

`__GNUC__` (e.g. 5) and `__GNUC_MINOR__` (e.g. 1).

To check that this is gcc compiler version 5.1 or greater:

```
#if defined(__GNUC__) && (__GNUC___ > 5 || (__GNUC__ == 5 && __GNUC_MINOR__ >= 1))
// this is gcc 5.1 or greater
#endif
```

Notice the chack has to be: `major > 5 || (major == 5 && minor >= 1)`. If you only do `major == 5 && minor >= 1`, it won’t work for version 6.0.

### clang

```
__clang_major__`, `__clang_minor__`, `__clang_patchlevel__
```

### Visual Studio

`_MSC_VER` and `_MSC_FULL_VER`:

```
VS                        _MSC_VER   _MSC_FULL_VER
1                         800
3                         900
4                         1000
4                         1020
5                         1100
6                         1200
6 SP6                     1200    12008804
7                         1300    13009466
7.1 (2003)                1310    13103077
8 (2005)                  1400    140050727
9 (2008)                  1500    150021022
9 SP1                     1500    150030729
10 (2010)                 1600    160030319
10 (2010) SP1             1600    160040219
11 (2012)                 1700    170050727
12 (2013)                 1800    180021005
14 (2015)                 1900    190023026
14 (2015 Update 1)        1900    190023506
14 (2015 Update 2)        1900    190023918
14 (2015 Update 3)        1900    190024210
15 (2017 Update 1 & 2)    1910    191025017
15 (2017 Update 3 & 4)    1911
15 (2017 Update 5)        1912
```

More information:

- https://blogs.msdn.microsoft.com/vcblog/2016/10/05/visual-c-compiler-version/
- https://blogs.msdn.microsoft.com/vcblog/2017/11/15/side-by-side-minor-version-msvc-toolsets-in-visual-studio-2017/#comment-467525

### MinGW

MinGW (aka MinGW32) and MinGW-w64 32bit: `__MINGW32_MAJOR_VERSION` and `__MINGW32_MINOR_VERSION`

MinGW-w64 64bit: `__MINGW64_VERSION_MAJOR` and `__MINGW64_VERSION_MINOR`

## Checking processor architecture

### gcc

The meaning of those should be self-evident:

- `__i386__`
- `__x86_64__`
- `__arm__`. If defined, you can further check:
  - `__ARM_ARCH_5T__`
  - `__ARM_ARCH_7A__`
- `__powerpc64__`
- `__aarch64__`











