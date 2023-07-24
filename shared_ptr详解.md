[![img](https://zhuanlan.zhihu.com/p/547647844)
](javascript:void(0))





写文章

![点击打开dtfour07的主页](https://pica.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c)

# C++:共享指针shared_ptr的理解与应用

[![C语言资深大师](https://picx.zhimg.com/v2-15c5814a88804114c595e71418514b11_l.jpg?source=172ae18b)](https://www.zhihu.com/people/wo-xiang-zi-you-52)

[C语言资深大师](https://www.zhihu.com/people/wo-xiang-zi-you-52)

关注他

14 人赞同了该文章

## 一 为什么要使用shared_ptr？

在实际的 C++ 开发中，我们经常会遇到诸如程序运行中突然崩溃、程序运行所用内存越来越多最终不得不重启等问题，这些问题往往都是内存资源管理不当造成的。比如：

- 有些内存资源已经被释放，但指向它的指针并没有改变指向（成为了野指针），并且后续还在使用；
- 有些内存资源已经被释放，后期又试图再释放一次（重复释放同一块内存会导致程序运行崩溃）；
- 没有及时释放不再使用的内存资源，造成内存泄漏，程序占用的内存资源越来越多。

> 智能指针shared_ptr 是存储动态创建对象的指针，其主要功能是管理动态创建对象的销毁，从而帮助彻底消除内存泄漏和悬空指针的问题。

## 二 shared_ptr的原理和特点

**基本原理:**就是记录对象被引用的次数，当引用次数为 0 的时候，也就是最后一个指向该对象的共享指针析构的时候，共享指针的析构函数就把指向的内存区域释放掉。

**特点:**它所指向的资源具有共享性，即多个shared_ptr可以指向同一份资源，并在内部使用引用计数机制来实现这一点。

共享指针内存：每个 shared_ptr 对象在内部指向两个内存位置：

- 指向对象的指针；
- 用于控制引用计数数据的指针。

1.当新的 shared_ptr 对象与指针关联时，则在其构造函数中，将与此指针关联的引用计数增加1。

2.当任何 shared_ptr 对象超出作用域时，则在其析构函数中，它将关联指针的引用计数减1。如果引用计数变为0，则表示没有其他 shared_ptr 对象与此内存关联，在这种情况下，它使用delete函数删除该内存。

shared_ptr像普通指针一样使用，可以将*和->与 shared_ptr 对象一起使用，也可以像其他 shared_ptr 对象一样进行比较;

## 三 shared_ptr的使用

### 3.1.构造函数创建

```text
1.shared_ptr<T> ptr;//ptr 的意义就相当于一个 NULL 指针
2.shared_ptr<T> ptr(new T());//从new操作符的返回值构造
3.shared_ptr<T> ptr2(ptr1);    // 使用拷贝构造函数的方法，会让引用计数加 1
                               //shared_ptr 可以当作函数的参数传递，或者当作函数的返回值返回，这个时候其实也相当于使用拷贝构造函数。
4./*假设B是A的子类*/
shared_ptr<B> ptrb(new B());
shared_ptr<A> ptra( dynamic_pointer_cast<A>(ptrb) );//从 shared_ptr 提供的类型转换 (cast) 函数的返回值构造
5./* shared_ptr 的“赋值”*/
shared_ptr<T> a(new T());
shared_ptr<T> b(new T());
a = b;  // 此后 a 原先所指的对象会被销毁，b 所指的对象引用计数加 1
		//shared_ptr 也可以直接赋值，但是必须是赋给相同类型的 shared_ptr 对象，而不能是普通的 C 指针或 new 运算符的返回值。
		//当共享指针 a 被赋值成 b 的时候，如果 a 原来是 NULL, 那么直接让 a 等于 b 并且让它们指向的东西的引用计数加 1;
		// 如果 a 原来也指向某些东西的时候，如果 a 被赋值成 b, 那么原来 a 指向的东西的引用计数被减 1, 而新指向的对象的引用计数加 1。
6./*已定义的共享指针指向新的new对象————reset()*/
shared_ptr<T> ptr(new T());
ptr.reset(new T()); // 原来所指的对象会被销毁
```

### 3.2.make_shared辅助函数创建

```text
std::shared_ptr<int> foo = std::make_shared<int> (10);
```

**建议使用make_shared的方式构造**

### 3.3 自定义所指堆内存的释放规则

在初始化 shared_ptr 智能指针时，还可以自定义所指堆内存的释放规则，这样当堆内存的引用计数为 0 时，会优先调用我们自定义的释放规则。

在某些场景中，自定义释放规则是很有必要的。比如，对于申请的动态数组来说，shared_ptr 指针默认的释放规则是不支持释放数组的，只能自定义对应的释放规则，才能正确地释放申请的堆内存。

对于申请的动态数组，释放规则可以

使用 C++11 标准中提供的 default_delete 模板类

可以自定义释放规则

```text
//指定 default_delete 作为释放规则
std::shared_ptr<int> p6(new int[10], std::default_delete<int[]>());
//自定义释放规则
void deleteInt(int*p) {
    delete []p;
}
//初始化智能指针，并自定义释放规则
std::shared_ptr<int> p7(new int[10], deleteInt);
```

## 四 shared_ptr常用函数

- get()函数，表示返回当前存储的指针（就是被shared_ptr所管理的指针） 。
  但是不建议使用get()函数获取 shared_ptr 关联的原始指针，因为如果在 shared_ptr 析构之前手动调用了delete函数，会导致错误

```text
shared_ptr<T> ptr(new T());
T *p = ptr.get(); // 获得传统 C 指针
```

- use_count()函数，表示当前引用计数

```text
shared_ptr<T> a(new T());
a.use_count(); //获取当前的引用计数
```

- reset()函数，表示重置当前存储的指针

```text
shared_ptr<T> a(new T());
a.reset(); // 此后 a 原先所指的对象会被销毁，并且 a 会变成 NULL
```

- operator*，表示返回对存储指针指向的对象的引用。它相当于：* get（）。
- operator->，表示返回指向存储指针所指向的对象的指针，以便访问其中一个成员。跟get函数一样的效果。

**示例1：shared_ptr的基础应用**：

```text
#include <iostream>
#include  <memory> // 共享指针必须要包含的头文件
using namespace std;
int main()
{
	// 最好使用make_shared创建共享指针，
	shared_ptr<int> p1 = make_shared<int>();//make_shared 创建空对象，
	*p1 = 10;
	cout << "p1 = " << *p1 << endl; // 输出10

	// 打印引用个数：1
	cout << "p1 count = " << p1.use_count() << endl;
	
	// 第2个 shared_ptr 对象指向同一个指针
	std::shared_ptr<int> p2(p1);
	
	// 输出2
	cout << "p2 count = " << p2.use_count() << sendl;
	cout << "p1 count = " << p1.use_count() << endl;

	// 比较智能指针，p1 等于 p2
	if (p1 == p2) {
		std::cout<< "p1 and p2 are pointing to same pointer\n";
	}
	
	p1.reset();// 无参数调用reset，无关联指针，引用个数为0
	cout << "p1 Count = " << p1.use_count() << endl;
	
	p1.reset(new int(11));// 带参数调用reset，引用个数为1
	cout << "p1 Count = " << p1.use_count() << endl;

	p1 = nullptr;// 把对象重置为NULL，引用计数为0
	cout << "p1  Reference Count = " << p1.use_count() << endl;
	if (!p1) {
		cout << "p1 is NULL" << endl; // 输出
	}
	return 0;
}
```

**示例2：shared_ptr作返回值**

```text
shared_ptr<string> factory(const char* p) 
{
    shared_ptr<string> p1 = make_shared<string>(p);
    return p1;
}

void use_factory() 
{
    shared_ptr<string> p = factory("helloworld");
    int num1 = p.use_count();
    cout << *p << endl;//!离开作用域时，p引用的对象被销毁。
}
shared_ptr<string> return_share_ptr()
{
    shared_ptr<string> p = factory("helloworld");
    cout << *p << endl;
    return p; //!返回p时，引用计数进行了递增操作。 
} //!p离开了作用域，但他指向的内存不会被释放掉。 

int main()
{
    use_factory();
    auto p = return_share_ptr();
    cout << p.use_count() << endl;
    system("pause");
    return 0;
}
//可以认为每个shared_ptr都有一个关联的计数器，通常称其为引用计数。无论何时我们拷贝一个shared_ptr，计数器都会递增。
//例如，当用一个shared_ptr去初始化另一个shared_ptr；当我们给shared_ptr赋予一个新的值或者是shared_ptr被销毁(例如一个局部的shared_ptr离开其作用域)时，计数器就会递减。
//一旦一个shared_ptr的计数器变为0，他就会自动释放自己所管理的对象。
```

**示例3：容器中的shared_ptr-记得用erease节省内存**

对于一块内存，shared_ptr类保证只要有任何shared_ptr对象引用它，他就不会被释放掉。由于这个特性，保证shared_ptr在不用之后不再保留就非常重要了，通常这个过程能够自动执行而不需要人工干预，有一种例外就是我们将shared_ptr放在了容器中。所以永远不要忘记erease不用的shared_ptr。

```text
#include <iostream>
using namespace std;
int main()
{
    list<shared_ptr<string>>pstrList;
    pstrList.push_back(make_shared<string>("1111"));
    pstrList.push_back(make_shared<string>("2222"));
    pstrList.push_back(make_shared<string>("3333"));
    pstrList.push_back(make_shared<string>("4444"));
    for(auto p:pstrList)
    {
        if(*p == "3333");
        {
            /*do some thing!*/
        }
        cout<<*p<<endl;
    }
    /*包含"3333"的数据我们已经使用完了！*/
    

    for(list<shared_ptr<string>>::iterator itr = pstrList.begin();itr!=pstrList.end();++itr)
    {
        if(**itr == "3333"){
            cout<<**itr<<endl;
            pstrList.erase(itr);
        }
    }
    cout<<"-------------after remove------------"<<endl;
    for(auto p:pstrList)
    {
        cout<<*p<<endl;
    }
　　while(1)
　　{
　　　　/*do somthing other works!*/
　　　　/*遍历 pstrList*/    //！这样不仅节约了大量内存，也为容器的使用增加了效率　　
　　}
 }
```

**示例4：shared_ptr：对象共享相同状态**

> 使用shared_ptr在一个常见的原因是允许多个多个对象共享相同的状态，而非多个对象独立的拷贝！

```text
#include <iostream>
using namespace std;
void copyCase()
{
    list<string> v1({"1","b","d"});
    list<string> v2 = v1;        //!v1==v2占用两段内存

    v1.push_back("cc");            //!v1!=v2

    for(auto &p:v1){
        cout<<p<<endl;
    }
    cout<<"--------void copyCase()---------"<<endl;
    for(auto &p:v2){
        cout<<p<<endl;
    }
} //v1和v2分属两个不同的对象，一个改变不会影响的状态。
void shareCase()
{
    shared_ptr<list<string>> v1 = make_shared<list<string>>(2,"bb");
    shared_ptr<list<string>> v2 = v1;

    (*v1).push_back("c2c");
    for(auto &p:*v1){
        cout<<p<<endl;
    }
    cout<<"----------shareCase()--------"<<endl;
    for(auto &p:*v2){
        cout<<p<<endl;
    }
} //v1和v2属于一个对象的两个引用，有引用计数为证，其内容的改变是统一的。

int main()
{
    copyCase();
    cout<<"++++++++++++++++"<<endl;
    shareCase();
}
```

**示例5：shared_ptr管理动态数组**

默认情况下，shared_ptr指向的动态的内存是使用delete来删除的。这和我们手动去调用delete然后调用对象内部的析构函数是一样的。与unique_ptr不同，shared_ptr不直接管理动态数组。如果希望使用shared_ptr管理一个动态数组，必须提供自定义的删除器来替代delete 。

```text
#include <iostream>
using namespace std;

class DelTest
{
public:
    DelTest(){
        j= 0;
        cout<<" DelTest()"<<":"<<i++<<endl;
    }
    ~DelTest(){
        i = 0;
        cout<<"~ DelTest()"<<":"<<i++<<endl;
    }
　　static int i,j;
};
int DelTest::i = 0;
int DelTest::j = 0;
void noDefine()
{
    cout<<"no_define start running!"<<endl;
    shared_ptr<DelTest> p(new DelTest[10]);

}
void slefDefine()
{
    cout<<"slefDefine start running!"<<endl;
    shared_ptr<DelTest> p(new DelTest[10],[](DelTest *p){delete[] p;});
}     　　　　　　　　　　　　　　　　//!传入lambada表达式代替delete操作。
int main()
{
    noDefine(); 　　//!构造10次，析构1次。内存泄漏。
    cout<<"----------------------"<<endl;
    slefDefine();  //!构造次数==析构次数 无内存泄漏
}
```

## 五 注意

### 5.1.常见错误(注意以下代码全是错误代码)

**1.不能使用原始指针初始化多个shared_ptr。**

```text
int* p11 = new int;
std::shared_ptr<int> p12(p11);
std::shared_ptr<int> p13(p11);
// 由于p1和p2是两个不同对象，但是管理的是同一个指针，这样容易造成空悬指针，
//比如p1已经将aa delete了，这时候p2里边的aa就是空悬指针了
```

**2.不允许以暴露裸漏的指针进行赋值**

```text
//带有参数的 shared_ptr 构造函数是 explicit 类型的，所以不能像这样
std::shared_ptr<int> p1 = new int();//不能隐式转换，类型不匹配
```

隐式调用它构造函数

**3.不要用栈中的指针构造 shared_ptr 对象**

```text
int x = 12;
std::shared_ptr<int> ptr(&x);
```

shared_ptr 默认的构造函数中使用的是delete来删除关联的指针，所以构造的时候也必须使用new出来的堆空间的指针。当 shared_ptr 对象超出作用域调用析构函数delete 指针&x时会出错。

**4.不要使用shared_ptr的get()初始化另一个shared_ptr**

```text
Base *a = new Base();
std::shared_ptr<Base> p1(a);
std::shared_ptr<Base> p2(p1.get());
//p1、p2各自保留了对一段内存的引用计数，其中有一个引用计数耗尽，资源也就释放了,会出现同一块内存重复释放的问题
```

## 5. 多线程中使用 shared_ptr

shared_ptr的引用计数本身是安全且无锁的，但对象的读写则不是，因为shared_ptr 有两个数据成员，读写操作不能原子化。shared_ptr 的线程安全级别和内建类型、标准库容器、std::string 一样，即：

- 一个 shared_ptr 对象实体可被多个线程同时读取
- 两个 shared_ptr 对象实体可以被两个线程同时写入，“析构”算写操作
- 如果要从多个线程读写同一个 shared_ptr 对象，那么需要加锁

发布于 2022-07-29 16:34

[C / C++](https://www.zhihu.com/topic/19601705)

[指针（编程）](https://www.zhihu.com/topic/19636435)

[C++](https://www.zhihu.com/topic/19584970)

赞同 142 条评论

分享

喜欢收藏申请转载



![img](https://pica.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c)

评论千万条，友善第一条

2 条评论

默认

最新

[![Dreamboat](https://picx.zhimg.com/v2-09d2ee82284ec990aaef1e54c66e889d_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/0338eb7bc1fa1ab36200a10e26d371b9)

[Dreamboat](https://www.zhihu.com/people/0338eb7bc1fa1ab36200a10e26d371b9)



list迭代器那里，得itr==pstrList.erase(itr);吧

05-29 · IP 属地北京

回复赞

[![芒果柚](https://picx.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/d1900a8cf7a42a7aa23a86e652e7f2b3)

[芒果柚](https://www.zhihu.com/people/d1900a8cf7a42a7aa23a86e652e7f2b3)



讲的很好！要是示例代码的仿真结果也贴出来就更好啦

05-28 · IP 属地江苏

回复赞

### 推荐阅读

- # C++之 shared_ptr智能指针

- 在实际的 C++ 开发中，我们经常会遇到诸如程序运行中突然崩溃、程序运行所用内存越来越多最终不得不重启等问题，这些问题往往都是内存资源管理不当造成的。比如： 有些内存资源已经被释放，…

- 倔强的码农

- ![C++11 shared_ptr智能指针](https://pic1.zhimg.com/v2-0e1c63a4a448a31a1839f4ac39d4058d_250x0.jpg?source=172ae18b)

- # C++11 shared_ptr智能指针

- 张博的求真之路

- ![C++动态指针之shared_ptr](https://pica.zhimg.com/v2-3a3bf5f33d7ea7fe13a6bbc8fedc070f_250x0.jpg?source=172ae18b)

- # C++动态指针之shared_ptr

- 百度张峻维

- ![C++ shared_ptr的坑](https://pica.zhimg.com/v2-6ed851d27e43fe8bb42f3a769898c31b_250x0.jpg?source=172ae18b)

- # C++ shared_ptr的坑

- 百度张峻维