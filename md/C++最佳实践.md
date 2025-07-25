C++因其媲美C的性能和跨平台特性，被广泛应用于追求高性能的领域，如嵌入式系统、服务器开发、音视频处理、游戏引擎以及量化金融等。

然而，高性能的代价是较低的易用性。毕竟，高风险往往伴随着高回报。

与现代语言如`Go`、`C#`、`Rust`相比，C++的使用门槛更高。正如C++之父本贾尼·斯特劳斯特卢普所言：  
**“软件行业中，太多的管理者试图将编程变成低级别的流水线工作。从长远来看，这种做法效率低下、浪费严重、成本高昂，而且不人性化。在软件开发中，没有放之四海而皆准的模型。我们需要为人们提供发挥才干的空间，并鼓励他们成长。”**

C++的设计初衷正是为了**让开发者能够更加自由灵活地编程，而不受语言本身的限制**。  
本篇文档的编写旨在记录我发现的一些容易踩坑的点，希望能为他人提供参考。

---
### 兼容性问题

C++标准委员会每三年更新一次语言标准，但他们仅负责定义标准，具体的实现由各编译器厂商（如 MSVC、GCC、Clang 等）完成。因此，不同编译器生成的 C++ 代码可能互不兼容。例如，使用 MSVC 编译的 DLL 可能无法被 GCC 编译的代码调用：

```c++
// MSVC 编译的 DLL 导出的函数
__declspec(dllexport) void print_string(const std::string& s);

// GCC 编译的调用方
int main() {
    std::string s = "hello";
    print_string(s); // 可能崩溃！
}
```

上述代码可能崩溃的原因在于，MSVC 实现的 `std::string` 与 GCC/Clang 的实现不同（例如成员变量顺序或其他实现细节），导致 ABI 及内存布局不一致，从而引发崩溃。  
**统一工具链以避免此类问题的发生。**  

`win`与`linux`下动态库的路径搜索行为差异。   

**Windows 下 `.dll` 的搜索路径：**
1. 可执行文件（.exe）当前目录
2. 系统路径  
   - 64 位系统：`%SystemRoot%\System32`
   - 32 位兼容目录：`%SystemRoot%\SysWOW64`
3. `Path` 环境变量中指定的目录

**Linux 下 `.so` 的搜索路径：**
1. 通过 `-Wl,-rpath,<path>` 嵌入到 ELF 文件中的路径（**优先级最高**）
2. 用户临时指定的路径（如 `export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH`）
3. 系统默认路径  
   - `/lib`
   - `/usr/lib`
   - `/lib64`
   - `/usr/lib64`

linux下默认不从 `. `路径下进行.so搜索，需到`LD_LIBRARY_PATH` 或 `-rpath` 显式添加，在实际部署程序时可通过编写脚本将库安装到目标平台的系统路径下或是在编译时确定好编译选项。
推荐通过`dlopen`在运行时加载对应的`.so`：
```cpp
void* dl_handle = dlopen("./libcamera.so", RTLD_LAZY | RTLD_GLOBAL);
if(!dl_handle){
	std::cerr << "libcamera open err!\n";
    fprintf(stderr, "dlopen failed: %s\n", dlerror());
}
// 通过dlsym(dl_handle,..);获取指定符号的地址
dlclose(dl_handle);
```
win中对应的函数为`LoadLibraryEx`|`LoadLibrary`|`GetProcAddress`|`FreeLibrary`  

在win下使用`protobuf`时，需要在`.h`中显示添加宏`#define PROTOBUF_USE_DLLS`，表示以`dll`的形式导入其中的函数(`__declspec(dllimport)`),否则会发生链接错误(linux下通常不需要显示指定导入导出)。

---
### std::shared_ptr

考虑以下代码：

```c++
// 假设 func 用来执行相关操作并返回一个值
int func();

int main() {
    do_something(func(), std::shared_ptr<A>(new A));  // 潜在的资源泄漏
}
```

上述代码看似合理，却忽略了**指令重排**问题。在运行中，一个函数的实参必须先被计算，这个函数才会开始执行。所以在调用 `do_something` 之前，必须执行以下操作：
-  执行 `func()` 并获取返回值。
-  执行表达式 `new A`。
-  调用 `std::shared_ptr<A>` 的构造函数接管表达式 `new A` 返回的指针。

编译器并不保证按照执行顺序生成代码，因此可能会产生如下结果：
1. 执行表达式 `new A`。
2. 执行 `func()` 并获取返回值。
3. 调用 `std::shared_ptr<A>` 的构造函数。

如果 `func()` 在执行过程中产生异常，`new A` 返回的指针将永远不会被 `std::shared_ptr<A>` 的构造函数所接管，从而导致资源泄漏。

使用 `std::make_shared` 可以避免上述问题。它不仅更高效（只分配了一次内存，内存块同时容纳了被管理对象和控制块），还能确保资源管理的安全性：
```c++
int func();

int main() {
    do_something(func(), std::make_shared<A>());  // std::make_unique同理
}
```
> **注意**：`std::make_unique` 从 C++14 开始支持。当然可以手动实现一个来模仿`std::make_unique`的行为,这很简单：
```cpp
template<typename T,typename... As>
std::unique_ptr<T> test_make_unique(As&&... as){
	return std::unique_ptr<T>(new T(std::forward<As>(as)...));
}
```

也存在`std::make_shared`无法替代的情况，当需要自定义删除器时，`std::make_shared`无法做到：
```cpp
void delete_func(A* ptr);        // 自定义删除器
int main(){
	do_something(func(),std::shared_ptr<A>(new A,delete_func));
}
```
将`std::shared_ptr`放到自己的语句中，以避免上述问题的发生：
```cpp
void delete_func(A* ptr);        // 自定义删除器
int main(){
	std::shared_ptr<A> sptr(new A,delete_func);
	do_something(func(),sptr);
}
```

在使用标准库时难免会考虑线程安全的问题，`std::shared_ptr`同理，考虑以下代码：
```cpp
struct Base{};
struct Dervied : Base{};
void print(const std::shared_ptr<Base>& p);   // 做一些打印操作

void thr(std::shared_ptr<Base> p){
	std::this_thread::sleep_for(987ms);
	std::shared_ptr<Base> t = p;    // 线程安全，虽然自增引用计数
	{
		static std::mutex m;
		std::lock_guard<std::mutex> lock(m);
		print(t);
	}
}

int main(){
	std::shared_ptr<Base> sp = std::make_shared<Derived>();
	std::thread t1{thr,sp},t2{thr,sp},t3{thr,sp};
	sp.reset();       // 释放主线程的use_count

	t1.join();
	t2.join();
	t3.join();
}
```
多个`std::shared_ptr`所管理的是同一份数据，用的是同一个`use_count`，但是各自是不同的对象。当发生多线程中操作不同的`std::shared_ptr`的成员函数时，不会出现数据竞争(即使这些对象是副本并共享同一对象的所有权，也无需额外的同步)。
`std::shared_ptr`本身的`use_count`操作(即拷贝、赋值、析构)是线程安全的，也就是说多个线程可以同时拷贝同一个`std::shared_ptr`，其`use_count`会被正确维护。

>为了满足线程安全要求，引用计数器通常使用等效于 [std::atomic::fetch_add](https://cppreference.cn/w/cpp/atomic/atomic/fetch_add "cpp/atomic/atomic/fetch add") 和 [std::memory_order_relaxed](https://cppreference.cn/w/cpp/atomic/memory_order "cpp/atomic/memory order") 的操作来递增（递减需要更强的排序以安全地销毁控制块）。

但对同一个示例同时读写（如一个线程调用`std::shared_ptr<T>::reset()`,另一个线程调用其拷贝移动构造)并非线程安全，考虑以下代码：
```cpp
std::shared_ptr<A> sptr;
// 线程1
void func1(){
	sptr.reset(new A{});
}
//线程2
void func2(){
	auto sptr2 = sptr;
}
```
当线程1将`sptr`的`use_count`释放后，触发`~A()`,此时`sptr2`可能仍然保存的是以析构的对象的地址，从而产生UB！
`std::shared_ptr `的原子变量目的是保证不同 `std::shared_ptr `的并发安全，重点是“不同”。

>如果多个执行线程在没有同步的情况下访问同一个 `std::shared_ptr` 对象，并且任何这些访问使用 `shared_ptr` 的非 const 成员函数，则会发生数据竞争。使用`std::atomic<shared_ptr<T>>`或是施加其他同步操作避免以上情况的发生。`

---
### Strict Aliasing(严格别名)

严格别名规定了对变量的读写需为相似类型，但宽限至可加上`cv`修饰。二是特许的 `char`、`unsigned char` 和 `std::byte` 类型。
变量不止是存储在特定地址的值，从编译器角度来看，还具有以下性质：
- 存储期
- 生存期
- 类型
- 一个可选的名字  
一个变量的存储期可以是：`automatic、static、thread `和 `dynamic`。它们决定了存储的分配（allocate）与解分配（deallocate）的时机。而存储期本身通常是由说明符或者所在命名空间决定的。比方说局部的无说明符对象就具有自动（automatic）存储期：代码块开始时分配，代码块结束时解分配。
而变量的生存期则表示对该变量的访问合法性，在变量的生存期之外访问该变量则是UB，且变量的生存期在其存储期之内，换言之，**变量在构造时开始生存期，在析构时结束生存期**。
考虑以下代码：
```cpp
auto ptr = (std::string*)malloc(sizeof(std::string) * 10);

for(int i = 0; i < 10; i++){
	ptr[i] = std::to_string(i);  // UB
}
```
这段代码显然并未遵循标准，因为对象的生存期尚未开始，却对其进行了访问。
使用`placement new`以显式开始对象的生存期。
```cpp
auto ptr = (std::string*)malloc(sizeof(std::string) * 10);

for(int i = 0; i < 10; i++){
	new (ptr + i)std::string(std::to_string(i));   // OK!
}
```
注意，在C++20之前，平凡类型(比如提供`(int*)std::malloc(sizeof(int))`)存储同样需要显示开始生存期。
在C++20之后，
[隐式生存期类型](https://en.cppreference.com/w/cpp/named_req/ImplicitLifetimeType)的对象不再需要显式开始生存期。隐式生存期类型的约束比平凡类型略微宽松，通常要求是聚合类，或者至少有一个平凡构造函数和一个平凡析构函数。

一些操作可以对隐式生存期类型施加隐式创建对象：
- `std::malloc()`。
- `std::memcpy()` 或者 `std::memmove()`。
- `char[]`、`unsigned char[]` 和 `std::byte[]` 开始生存期。
- `operator new` 或者 `operator new[]`。

回到之前的例子，但这次换为int类型：
```cpp
auto ptr = (int*)malloc(sizeof(int) * 10);

for(int i = 0; i < 10; i++){
	ptr[i] = i;  // OK ! C++20起
}
```
把`int`替换为`std::string`后依旧需要显示开始生存期，因为其为非平凡类型。
#### std::launder

`std::launder`函数对指针进行清洗，就像洗钱一样，对指针的来源进行"清洗"以阻止编译器分析来源。
`std::launder`主要用于解决生存期问题，`std::launder(T* p)` 接收一个指向地址 X 的指针 p，返回仍在生存期内的位于地址 X 的对象的指针。也就是说，从旧对象的指针中获取新对象的指针。
该函数的典型用途是，对同一个地址进行`placement new`后，因为原有地址上的对象X已结束生存期，虽然指针指向的是同一个地址X，但不再是合法访问，此时要么使用 placement new 返回的新的指针，要么使用 `std::launder(p)` 以合法访问对象。  
示例1：
```cpp
X* p = new X{};
p->func();   // 执行一系列操作
p->~X();     // 执行析构，结束该变量的生存期
new(p) X{};
std::launder(p)->func();        // 或是使用 auto a = new(p) X{} 同理
```
示例2：
```cpp
// 示例来自：https://en.cppreference.com/w/cpp/utility/launder
#include <cassert>
#include <cstddef>
#include <new>

struct Base {
    virtual int transmogrify();
};

struct Derived : Base {
    int transmogrify() override {
        new(this) Base;
        return 2;
    }
};

int Base::transmogrify() {
    new(this) Derived;
    return 1;
}

static_assert(sizeof(Derived) == sizeof(Base));

int main() {
    // Case 1: the new object failed to be transparently replaceable because
    // it is a base subobject but the old object is a complete object.
    Base base;
    int n = base.transmogrify();
    // int m = base.transmogrify(); // undefined behavior
    int m = std::launder(&base)->transmogrify(); // OK
    assert(m + n == 3);

    // Case 2: access to a new object whose storage is provided
    // by a byte array through a pointer to the array.
    struct Y { int z; };
    alignas(Y) std::byte s[sizeof(Y)];
    Y* q = new(&s) Y{2};
    const int f = reinterpret_cast<Y*>(&s)->z; // Class member access is undefined
                                               // behavior: reinterpret_cast<Y*>(&s)
                                               // has value "pointer to s" and does
                                               // not point to a Y object
    const int g = q->z; // OK
    const int h = std::launder(reinterpret_cast<Y*>(&s))->z; // OK

    [](...){}(f, g, h); // evokes [[maybe_unused]] effect
}
```

回到标题，在实现类型双关时，通常会将一个对象解释为不同的类型，而类型双关通过“别名”来实现，即通过获取对象的地址，并将他转换为想重新解释的类型的指针，然后访问该值，**严格别名**则代表编译器会在一定规则下默认它们指向不同的内存区域（即使它们实际上指向相同的内存区域），并以此进行优化。
>gcc在 -O2,-O3,-Os下默认开启严格别名。GCC -O0, -O1 编译优化选项下开启严格别名（strict aliasing）规则的编译选项为：-fstrict-aliasing
```cpp
int test;
void func(float* a,int* b);

func((float*)(&test),&test); // 违背strict aliasing,编译器认为a,b指向不同的内存区域,为UB!
func((char*)(&test),&test); // 符合strict aliasing,编译器认为a,b指向同一内存区域
```
正如开头所说,`char`类型是严格别名规则下的银弹，可以作为任何类型的别名。不只是 `char` 类型，`unsigned char`，`uint8_t`, `int8_t`也满足这条规则。或是像[cppreference](https://en.cppreference.com/w/cpp/utility/launder)中的示例所示，使用`std::bytes(since C++17)`实现类型双关。

---
### 多线程

避免使用臭名昭著的双重检查锁`（Double-Checked Locking）`
考虑以下用双重检查锁实现的单例：
```cpp
class Singleton {
public:
    // 获取单例实例的静态方法
    static std::shared_ptr<Singleton> getInstance() {
        if (instance == nullptr) {  // 第一次检查（无锁）
            std::lock_guard<std::mutex> lock(mtx);  // 加锁
            if (instance == nullptr) {  // 第二次检查（加锁后）
                 instance.reset(new Singleton());
            }
        }
        return instance;
    }
    void doSomething() {
        std::cout << "Singleton instance is working!" << std::endl;
    }
private:
    Singleton() = default; 
    ~Singleton() = default;
    static std::shared_ptr<Singleton> instance;  
    static std::mutex mtx;  
};

std::shared_ptr<Singleton> Singleton::instance = nullptr;
std::mutex Singleton::mtx;
  
int main() {
    auto instance = Singleton::getInstance();
    instance->doSomething();
    return 0;
}
```
这段代码忽略了一些问题：
- 在`instance = std::make_shared<Singleton>();`的过程中，可能会先分配内存并将指针赋值给`instance`，然后再调用构造，若另一个线程在第一次检查后读取了`instance`，` instance->doSomething();`处则可能会访问一个未完全构造的对象，从而导致未定义的行为`(UB)`。
- `std::shared_ptr`没有做同步操作，在多线程的环境下同时访问可能导致数据竞争,引发未定义的行为。可以使用`std::atomic<std::shared_ptr<T>>`避免此问题。
- 虽然锁内的第二次检查能避免重复初始化，但第一次检查的读取操作（无锁）与写操作（加锁）之间仍存在数据竞争。

使用标准库的`std::call_once`以实现更安全的单例
```cpp
class Singleton {
public:
    static std::shared_ptr<Singleton> getInstance() {
	    static std::once_flag flag;
	    std::call_once(flag,[](){
		    instance.reset(new Singleton());
	    });
        return instance;
    }
private:
    Singleton() = default; 
    ~Singleton() = default;
    static std::shared_ptr<Singleton> instance;  
};
```
或是使用`magic static`,效果和`std::call_once`相同
```cpp
static Singleton& getInstance(){
	static Singleton s;
	return s;
}
```

---
### CRTP

在实现多态操作时，首选的方式是使用虚函数。
但虚函数存在的一定的运行时开销，每个包含虚函数的类都会维护一个`虚函数表(vtable)`，其存储了指向虚函数实现的指针。当调用虚函数时，程序需要通过对象的虚函数表指针查找对应的函数地址，这增加了一次间接寻址操作，相比普通的函数调用略慢。
且编译器无法对虚函数调用进行**编译期内联优化**，正如如上所说，虚函数调用在运行时根据对象的实际类型决定。
避免以上额外不必要的性能开销的手段之一就是采用[CRTP]([Curiously Recurring Template Pattern - cppreference.com](https://en.cppreference.com/w/cpp/language/crtp.html))(奇异递归模板模式)`实现编译期多态。
`CRTP`实现大致如下:
```cpp
template<typename Dervied>
class Base{
public:
	void print(){
		static_case<Dervied*>(this)->print();
	}
};

class Dervied : Base<Dervied>{
public:
	void print(){
		std::cout << " Dervied1::print() " <<std::endl;
	}	
};
```
`CRTP`减少了额外的运行时开销，正因为他是基于模版的多态，函数调用在编译期就确定，不存在虚函数表和动态绑定，编译器也可以直接在编译期进行内联展开，从而提高性能。  
`CRTP`的多态使用方式：
```cpp
template<T>
void func(Base<T>& base){
	base.print();
}

int main(){
	Dervied d;
	func(d);         // 输出 Dervied1::print()
}
```

在C++23标准下的显示对象成员函数时，`CRTP`纯在着坑: 
```cpp
template <typename Derived>
struct Base {
    void foo(this auto&& self) {
        std::cout << "self is " << typeid(decltype(self)).name() << "\n";
    }
};
struct Derived : Base<Derived> {};
struct FurtherDerived : Derived {};

auto main() -> int{
    FurtherDerived d;
    d.foo();    //gcc下输出self is 14FurtherDerived
} 
```
并非期望的`Derived`类型,更改为传统形式即可：
```cpp
void foo() {
    std::cout << "self is " << typeid(decltype(*static_cast<Derived*(this))).name() << "\n";
}
```

--- 
### 内存模型和原子操作

在理解内存模型之前，首先要了解什么是同步关系与先行关系。
#### 同步关系
**同步关系只存在于原子类型的操作之间。若一种数据结构含有原子类型，并且其整体操作都涉及恰当的内部原子操作，那么该数据结构的多次操作之间（如锁定互斥）就可能存在同步关系。但同步关系从根本上来说来自原子类型的操作。**
同步关系的基本思想是：对变量 x 执行原子写操作 W 和原子读操作 R，且两者都有适当的标记。只要满足下面其中一点，它们即彼此同步。
- R读取了 W 直接存入的值
- W 所属线程随后还执行了另一原子写操作，R 读取了后面存入的值
- 任意线程执行一连串“读-改-写”操作(如`fetch_add()或compare_exchange_weak()`)，而其中第一个操作读取的值由 W 写出。
#### 先行关系
先行关系和严格先行关系是在程序中确立操作次序的基本要素；它们的用途是清楚界定哪些操作能看见其他操作产生的结果。在单一线程中，这种关系非常直观：若某项操作按控制流程顺序在另一项之前执行，前者即先于后者发生，且前者严格先于后者发生。但如果同一语句内出现多个操作，则它们之间通常不存在先行关系，因为C++标准没有规定执行次序（换言之，执行次序不明）。比如之前所提到的一个坑：
```cpp
int func();
do_something(func(), std::shared_ptr<A>(new A));  // 潜在的资源泄漏(执行次序不明！)
```
以上规则实质上还是单线程次序执行，关键在于线程间的互动：*若某一线程执行甲操作，另一线程执行乙操作，从跨线程视角观察，甲操作先于乙操作发生，则甲操作先行于乙操作*。
在基操层面上，线程间先行关系相对简单，它依赖于前面所介绍的同步关系：若甲、乙两操作分别由不同线程执行，且它们同步，则甲操作跨线程的先于乙操作发生。这也是可传递关系：若甲操作跨线程的先行于乙操作发生，且乙操作跨线程的先行于丙操作发生，则甲操作跨线程的先行于丙操作发生。
以上规则都非常重要，它们强制线程间操作服从一定的次序。
#### 原子操作的内存次序
原子类型上的操作服从6钟内存次序：`memory_order_relaxed`、`memory_order_consume`、`memory_order_acquire'、`memory_order_release`、`memory_order_acq_rel` 和 `memory_order_seq_cst`。
其中,`memory_order_seq_cst`是可选的最严格的内存次序，各种原子类型的所有操作都默认遵循该次序，除非特意为某些操作

**先后一致次序**

---
### references
- 《C++并发编程实战(第二版)》
- 《Effective Modern C++》
- 《C++ Core Guidelines》
-    [Cppreference.](https://zh.cppreference.com/w/cpp)
