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



















