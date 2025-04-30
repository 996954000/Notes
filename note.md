## 编译

## 链接
- 链接器将一个个编译单元串起来

## 函数
- 函数的调用在编译后就是call操作，需要保存返回地址，将参数压入堆栈等等操作，并且涉及到内存的跳转，从而多余的函数调用操作也会降低程序性能

## 变量
- 变量不同仅在于其占用内存空间不同
- 有符号变量会将其中一bit拿出来用于标注符号
- 寻址的最小单位是byte，所以即使bool仅可使用1bit就行，但实际上还是用了1byte的空间（但有将8个bool放在一个byte中的操作）

## 头文件
- 头文件实际就是复制粘贴操作，里面我们往往可以将函数声明之类的放进去，然后四处include
- 头文件中默认加的#program once预处理操作用于防止头文件多次被引用导致复制粘贴，以往还有另一种方式，即预处理中使用信号量的方式来进行监管
- 头文件中<>和""区别仅仅在于<>是表示头文件在包含目录中，""则是用于指定头文件具体路径，一般都是相对路径，所以平常看起来像是自带的库用<>，自定义的用""，但实际上并不是

## 分支
- 实际上分支和函数的调用一样会是一个到处跳内存的操作，这意味它的性能也会很低，因此避免分支也是需要考虑的

## 指针
- 指针仅仅就是用于保存地址的变量，其中地址是个整数
- 在理解上指针的类型不重要，只是用来告诉编译器我这边占多大内存，所以在操作中，指针的类型又很重要，不然会发生危险操作
- 指针本身就是一个变量，同样保存在内存中（32位占4byte， 同理64位占8byte）
```c++
 int* p = 5; // 该声明是错误的，你在尝试将整数赋给一个指针变量
```

如果你想让 p 指向一个具体的整数值，你应该先声明一个整数变量，然后让 p 指向这个变量的地址，
```c++
 int num = 5;
 int* p = &num; // p 指向 num 的地址 
```

或者使用动态内存分配来获取一个可以被 p 指向的内存位置。
```c++
int *p = (int *)malloc(sizeof(int)); // 分配内存
*p = 5; // 将 5 存储在 p 指向的位置
free(p) // 释放
```
以下指针声明是等价的，空格不影响语义
```c++
int* p; // 指向整数的指针
int *p; // 同上
int * p; // 同上
int  *p; // 同上
```

以前可能会因为什么时候加*以及为什么加 *产生混淆
实际上`int* p`是声明了一个指针变量p，而声明之后的`*p = 5`中的`*`是反引用符号而不是指针声明用的`*`号，反引用是在访问指针 p 所指向的内存位置的内容。

`关于声明`: 单行声明多个时要用`int* a, *b`;后面跟上的变量的前面的*不要忘了


## 引用
- `&a`用于获取变量的地址， 这里的`&`为取地址符，`a`是已经定义的变量、
- `引用（无论是普通引用还是智能指针的引用）都不拥有对象的所有权。引用仅仅是另一种访问对象的方式，它们自身不拥有对象的所有权。"别名"终究是"别名"`
- `int& b = a`，这里的`&`是和`int`一体的，组成了`int&`，指的是引用，实际上就是给变量a起了个别名为b，在编译过程中会发现实际上不存在b这个变量，这可被视为只是用来使编程更为方便的语法糖
- 那么这个语法糖仅仅是说起别名吗，如果不使用引用这个东西，某程序如下
```c++

void addOne(int* num){
    (*num)++;
}

int main(){
    int a = 2;
    addOne(&a)
}

```

使用引用的话，它们在编译器里所做的事是一样的，所以引用可以在某些场景中代替以往指针的引用传值，这样可以使程序可读性更好，更简洁

```c++

void addOne(int& num){
    num++;
}

int main(){
    int a = 2;
    addOne(a)
}

```
- 需要注意的是引用在指向一个变量后无法二次更改，且一定需要初始化，这些特性使得我们在能使用引用的场景就尽量使用引用，这会更加的安全，并且可读性高
- `个人理解`，实际上也说了引用是指针的语法糖，但它不占用空间，并且在编译器中会发现比如`int a = 5; int& ref = a;`这里的ref在编译器中是不存在的，我们只有一个变量a，这相当可以理解为我将这一块内存起了两个名字，而这个名字是和a隐式的挤在一起，这块空间既可以叫a又叫ref，这样是不占空间的；但是传统的指针不同，我们需要为指针开辟一块空间，并将指针指向这块地址，才能在后续通过指针找到这块内存再进行操作，这里涉及到地址的跳转，查找以及新建指针等操作，自然也就慢了
- 多用引用，比如传类时之类的

## 堆和栈
- 程序中主要需要注意到的就是堆区和栈区
- 栈区的存储系统基本只需要维护一个栈指针，像平时int i = 1这种局部变量的操作都是在栈区中申请内存，栈区申请内存会是一系列连续的空间，比如上一行申请的i和下一行申请的j的内存空间是相邻的（debug模式中会空几位出来防止溢出，也就是安全守卫），而系统在分配内存时只需要维护一个指针栈指针就行了，当这些局部变量的作用域结束时，系统直接将栈中该作用域的内存中的东西全部弹出，结束。
- 堆区则是我们在new 时所分配的区域，new命令下会调用malloc操作，即动态分配内存，该操作会涉及到一系列的在空闲列表中寻找合适的内存块，保存内存块信息等操作，自然这些东西会拖慢性能，自然堆中存储的变量所占用的内存不像栈区中那样是连续的，并且需要用户去注意手动销毁等操作，堆中的数据往往需要跨作用域使用，并且数据很大的时候一般也是存堆里，栈的初始分配大概只有2m左右
- 还有一些比如cache miss等小知识自己看视频回顾
- 栈中由于是一条线上的内存连续存储，所以变量紧挨着变量，这意味着它们这一条缓存线大概率会同时被读入到缓存，cpu等高速存储空间中，这样在访问相近的或是相邻的变量时会比堆中的变量不停的在低速存储区中找要快很多（也是cache miss那块意思）
- `栈和堆速度间的区别最大体现于分配过程上，因为cache miss那块的问题如果不是百万级的数据的话可能感受不出来`

## 结构体和类
- 实际上结构体和类在语义上并没有什么区别，`这句话挺重要的，这意味着结构体的行为和类基本一致，可以被继承，可以添加虚函数，析构函数等操作`
- 结构体会将成员默认为公开，`类默认为私有,类默认为私有,类默认为私有`
- 一般在建立一个简单数据类时可以用结构体，比如坐标，向量
- 当需求需要涉及到一系列的继承关系等，则就用类
- 结构体中没有任何成员时其本身占一个byte用于寻址，如果有成员变量则结构体的大小为所有成员变量大小的和，此时寻址的工作交给首个成员变量的地址就行。
- `匿名结构体不会产生多态等一堆特性，它仅仅是一个数据结构，从而匿名结构体才能在匿名联合体中使用`

## Static
-  `Static`在类或结构体外时，表示该类或结构体仅为当前翻译单元可见，链接仅在内部
-  在类之内时，表示该静态变量或方法与所有该类的实例共享
    ### 类或结构体外Static
    - 如上所说表示仅当前翻译单元可见
    - 当在不同的文件中定义两个相同名称的全局变量时会无法编译，因为连接起来后会冲突，此时要么将其中一个加上static，要么就是删除其中之一的变量初始化使用`extern`关键字
    ### extern
    - 该关键字表示声明一个在外部定义的变量或函数
    - 该关键字不能在函数块中使用，函数块中的变量或函数时为局部变量（函数），这种变量或函数与某个外部定义的变量或函数这个概念矛盾，所以C++中不允许这种用法
    - `始终注意使用全局变量是一种灾难，要尽量避免；通过在类中使用静态是更好的，毕竟它们有作用域的限制`
    ### 类或结构体内Static
    - 该静态变量或方法与所有该类的实例共享
    - 静态方法无法使用非静态变量，因为静态方法没有类实例，实际上类中的非静态方法在使用时都会传入一个实例，该实例参数被隐藏了，而静态方法不会传入这个实例，从而也无法得知非静态成员的信息
    - 该静态成员的生命周期会和程序一致

## 构造函数和析构函数
- 构造函数就是在创建实例时调用的函数 `与方法同名的函数`
- 析构函数就是实例销毁时调用的函数 `在构造函数前面加个~`
- `不要在构造函数和析构函数前面加返回类型啥的`
## 继承

```c++
class M_parentClass {
public:
    int x, y;
    
    M_parentClass() {
        x = 0;
        y = 0;
    }
};

class M_subClass : public M_parentClass{
public:
    M_subClass() {
        x = 10;
        y = 10;
    }

    ~M_subClass() {
        std::cout << "调用析构函数";
    }
};


int main()
{
    M_subClass m;
    std::cout << m.x;
}
```

## 可见性
- `privat`: 私有
- `protect`: 该类以及其子类等可访问
- `public`: 公开

## 数组（原始风格数组）
- 这边是C++的原始数组
- 原始数组如果不借用sizeof的方法进行计算的话无法通过常规手段长度计算，所以维护数组需要靠自己，而标准库里的array当然会提供很多方法，当然这也就代表原始数组属于快速而又危险的工具。
- 在栈中或堆中（new）创建不同在于生命周期
- 数组名是指针
- [c++风格数组](#stl静态数组c风格数组)

## 字符串字面量
- `"abcd"`，这种用双引号括起来的字符串都是字符串字面量
- 字符串字面量默认就是 const char[] 类型
- `int s[5] = "abcd"`，其中s是在`栈区`的数组，并使用"abcd"这个字面量初始化，字面量存放在静态常量区，相当于s是静态常量区中那个字面量的一份`副本`
```c++
// 指向静态常量区的指针不可以进行修改值操作
char* s = (char*)"abcd";
    s[2] = 'b';
// “副本”可以进行修改操作
    char a[3] = "ab";
    a[1] = 'c';
```

## 字符串
- 编码：由于一些语言使用1byte的8bit不够进行符号编码，所以出现utf-8等编码
- 同样的，标准库提供了字符串的高级版本`std::string`
- 关于字符串相加，该方法不可行，因为是将两个const char[]相加
```c++
std::string = "ab" + "cd";
```
- 可以通过标准库中的string进行重载实现相加
```c++
std::string = std::string("ab") + "cd";
```
- 标准库字符串类型提供很多工具，看文档查找

## const
- 在变量中
```c++
// 用于声明不对该变量进行修改
const int x;

// 指针指向的地址中的内容不可修改(const 在*前面)
const int* p;
int const* p;

// 指针指向的地址不可修改(const在*后面)
int* const p;

```

- 在类中，添加在方法后面的const表示该方法中不会对类进行修改操作
```c++
class Entity{
private:
    int X, Y;
public:
    int getX() const {
        X = 2; // 报错
        return X;
    }
}
```

## mutable
- 上文说到类中方法后的const约定不会对类做出修改操作
```c++
class Entity{
private:
    mutable int X, Y;
public:
    int getX() const {
        X = 2; // 对具有mutable关键字的变量，可以在const方法中进行修改，一般用于调试，平时不这么用
        return X;
    }
}
```

## 成员初始化列表
- 成员初始化列表是C++中为了初始化成员变量提供的操作，且`只能用于构造函数， 当然拷贝构造函数也是构造函数`
```c++
class Entity {
private:
    int X, Y;
public:
    //初始化成员列表
    Entity() : X(1), Y(2) {
    
    }

    Entity(int X) {

    }

    int getX() {
        return X;
    }

    int getY() {
        return Y;
    }
};
```
- 该方式在代码表层可以将构造函数中初始化的逻辑和构造函数中的其他业务逻辑分开，提高可读性
- 从底层看，原始的赋值初始化方式虽然在原始类型和复合类型（指针，引用（这个还是要用初始化成员列表的））上与使用成员列表初始化性能没有差距
- `但是对于类类型，使用成员初始化列表的方式初始化性能更快`，如下所示，在`m_Test`类中我们声明了一个Entity类，如果使用原始的赋值方法进行初始化，我们会首先调用无参构造方法创建一个Entity对象（`类中的类成员变量在进入该类的构造函数前会事先被创建`），再使用有参构造方法创建一个Entity对象并覆盖掉原先无参创建的对象，这在过程中我们创建了两个对象，性能很低
- 使用初始化成员列表其实感觉像是等价于在成员变量定义的时候直接初始化的感觉，但可能理解不到位，后期再说，好像有说法是初始化列表能`保证`在构造函数之前完成初始化，难不成在成员定义时直接初始化没法保证在构造函数之前完成？
- `初始化成员列表使用时顺序要与成员变量定义顺序一致`
```c++
class m_Test {
    Entity e;
public:
    // 初始化成员列表
    m_Test(): e(8){
    }

    // 原始赋值方法
    m_Test() {
        e = Entity(8);
    }
};
```

## 三元运算符
- x ? y : z 不提了，c里就有

## 创建对象和 new 关键字
- 从上面知道创建对象分为堆里和栈里
- `int i = 1` 栈创建对象
- `int* i = new int` 堆中创建对象
- 重点说下为什么是`int*`，始终记住一点，new的底层是在调用malloc，并返回了一个指向该内存的`void`指针（指针的类型仅作为操作的参考），既然返回的是指针，自然用指针接收
- new一个对象时过程包括从空闲列表中查找内存块，分配空间，返回指针，调用构造函数等一系列操作
- new 的对象务必记得`delete`，new 的对象务必记得`delete`，new 的对象务必记得`delete`，new 的对象务必记得`delete`，new 的对象务必记得`delete`，new 的对象务必记得`delete`，new 的对象务必记得`delete`，new 的对象务必记得`delete`
- new int[50] 这种对应`delete[]`
### replacement new 
- 简单提一嘴 `int* j = new(i) int`，在i的内存上开辟空间，相当于省区delete直接覆盖一手

## 虚函数与纯虚函数
```c++
class Base {
private:

public:
    virtual void printName() {
        std::cout << "base" << std::endl;
    }
};

class Entity : public Base {
public:
    void printName() override {
        std::cout << "entity" << std::endl;
    }
};

void test(Base* b) { // ver - 1
    b->printName();
}

int main()
{
    /* 有虚函数输出：  无虚函数输出：
    entity            entity
    entity            base
    entity            base
    */
    Entity e;
    e.printName();

    Entity* ep= &e;
    test(ep);

    Base* bp = new Entity();
    bp->printName();
}
```
- virtual表示该函数为虚函数，若不使用虚函数，则代码中的ep调用函数时由于本身是Base类指针，它本身是Entity类，但被转为Base后会自动调用当前的类的方法即Base的同名方法，虚函数可以帮助它去调用它们本来的类中的方法
- 虚函数配合override使用更好，比如可以名称检查等
### 纯虚函数
- 相当于接口， 即没有函数体的虚函数，所有子类必须实现该方法（和Java里的接口一样），其意义在于用于确保类有此方法
```c++
class Base {
private:

public:
    virtual void printName() = 0;
};
```
- `虚函数表`，虚函数之所以能准确定位在于虚函数表，虚函数表存放于静态常量区，同时类需要维护一个虚函数指针指向对应的虚函数表，虚函数指针则是存放于对象声明的区域（堆或栈），并且为了保证性能，虚函数指针存放在类的`最前面`
- 当一个类被作为基类时，我们需要将它的析构函数写为虚函数，如在`ver-1`中delete，若基类的析构函数未被定义为虚函数，则只会释放基类的资源
- `而构造函数不可以作为声明为虚函数，当我们构造一个子类的对象时，先调用基类的构造函数，构造子类中基类部分，子类还没有构造，还没有初始化，如果在基类的构造中调用虚函数，如果可以的话就是调用一个还没有被初始化的对象，那是很危险的`，所以C++中是不可以在构造父类对象部分的时候调用子类的虚函数实现。但是不是说你不可以那么写程序，你这么写，编译器也不会报错。只是你如果这么写的话编译器不会给你调用子类的实现，而是还是调用基类的实现。
- `有趣的是我们将构造和析构两个过程放一起看，可以发现子类的构造加析构函数会是一个一层层的走法，调用基类构造->子类构造->子类析构->基类析构`，这样就可以很明显的理解为什么析构函数要为虚函数而构造函数不能为虚函数

## 隐式转换与explicit关键字
- C++中会默认允许进行`一次`隐式类型转换，但只允许一次，弯弯绕绕的有点不好解释
- explicit关键字则是限制这种转换，在智能指针中用到

## C++运算符重载
- 运算符重载顾名思义，这东西只有C++里有
```c++
class Vector {
public:
    float X, Y;

    Vector() {

    }

    Vector(float X, float Y) : X(X), Y(Y) {

    }

// 提供函数版本
    Vector add(Vector& other) {
        Vector tempVector(other.X + this->X, other.Y + this->Y);
        return tempVector;
    }
// 提供运算符重载版本
    Vector operator+ (Vector& other) {
        return add(other);
    }
};

int main()
{
    Vector v1(1.1, 1.1);
    Vector v2(2.2, 2.2);

    Vector v = v1 + v2;
}
```
- 还有一个是标准库输出中 << 符号的重载，有点像Java里的重写toString，好像一般都是定义为一个单独的函数并作为友元
```c++
std::ostream& operator<< (std::ostream& stream, Vector v) {
    stream << v.X << "," << v.Y;
    return stream;
}

std::cout << v << std::endl; // 3.3, 3.3
```

## this关键字
- `this是当前实例对象的指针`，所以this->a是a变量而不是a指针
- 在const成员函数中this会自动转换为const指针

## 智能指针
- 先给一个简单的实现自动管理实例生命周期的例子
```c++
class Vector {
public:
    float X, Y;

    Vector() {
        std::cout << "creat Vector with NULL" << std::endl;
    }

    Vector(float X, float Y) : X(X), Y(Y) {
        std::cout << "creat Vector with XY" << std::endl;
    }

    ~Vector() {
        std::cout << "delete Vector" << std::endl;
    }
};

class MyPointer {
public:
    Vector* v;

    MyPointer() {

    }

    MyPointer(Vector* v) : v(v){

    }

    ~MyPointer() {
        delete v;
    }
};

int main()
{
    {
        // 通过栈类出作用域自动调用析构函数的特性来管理堆类的内存
        // 相当于堆类的指针外面套了个栈类
        MyPointer mp(new Vector(1.1, 1.1)); 
    }
}
```

- 那么智能指针也是类似的原理
- 记得包含库`#include <memory>`
### unique_ptr
- `std::unique_ptr<Vector> up = std::make_unique<Vector>(Vector(1.1, 1.1));`

- 很明显这是个栈对象指针，它不能被复制，因为它的拷贝构造函数和拷贝构造运算符都被删掉了以防意外，如果它被复制，那么其中一个指针b被释放时，则指针a仍然指向这个已经被释放的内存，这很显然是很危险的。
- `为什么可以引用，引用指向同一个地方不会造成以上问题吗`
```c++
int main()
{
    std::unique_ptr<Vector> up = std::make_unique<Vector>(Vector(1.1, 1.1));
    {
        std::unique_ptr<Vector>& up2 = up;
        up2.release();
    }
    std::cout << up.get()->X;
    

    std::shared_ptr<Vector> sp = std::make_shared<Vector>(Vector(1.1, 1.1));

    std::weak_ptr<Vector> wp;
    wp = sp;
}
```
- `引用并不会产生以上问题，需要明确的是up独占Vector对象的所有权并完全管理其生命周期，引用没有权利去管理，即使调用release方法，内存仍然不会被释放，且出作用域后up2无效
- `引用（无论是普通引用还是智能指针的引用）都不拥有对象的所有权。引用仅仅是另一种访问对象的方式，它们自身不拥有对象的所有权。`
- unique_ptr基本没什么开销，速度很快
### shared_ptr
- std::shared_ptr<Vector> sp = std::make_shared<Vector>(Vector(1.1, 1.1));
- 采用引用计数，该指针会自动记录被引用数，只有所有的引用释放后该指针指向的内存地址会被释放
- `std::shared_ptr<Vector> sp = std::make_shared<Vector>(Vector(1.1, 1.1));`
### weak_ptr
- `std::weak_ptr<Vector> wp;`该指针引用shared_ptr时不会增加其引用计数
- 也不用担心空指针问题，当shared_ptr被释放时，可以通过手动调用方法询问weak_ptr是否过期来避免空指针异常

## 拷贝构造函数
### 浅拷贝
- `Tip：使用 = 号时你永远在进行拷贝操作`
- `int a = 1; int b = a`，这种操作就是浅拷贝操作，b复制了a的值，对于基础类型的拷贝看起来没啥，但是对于引用类型`int* a = new int; int* b = a`，我们会发现它`仅仅会拷贝指针的值，也就是地址`，这样看好像并没有什么不对，代码如下所示
```c++
struct Vector
{
    int X, Y, Z;
};

class Entity {
public:
    Vector* v;
    std::string name;

    Entity(Vector* v, const std::string& name) : v(v), name(name) {

    }

    ~Entity() {
        delete v;
    }
};

int main()
{
    Vector v{1,2,3};
    
    Entity a(&v, "a");
    Entity b = a;
}
```
- 以上代码会引发多次释放同一个地址的内存的bug，原因就在于在进行拷贝时是浅拷贝操作，b实例中的Vector指针和a中的Vector指向同一个地址，从而导致在释放时会重复释放
- 我们实际的需求实际上应该是每个实例维护自己的Vector指针，也就是说我们每个实例中的引用变量需要指向不同的地址，需要的操作是在进行拷贝时将引用变量所指地址的值拷贝并新开一个空间拷贝该值，于是我们需要深拷贝
### 深拷贝
- 深拷贝就是将引用变量所指地址的值拷贝下来并再开一个空间存放该值，`深拷贝我们需要用到拷贝构造函数`，这块直接和动态数组Vector合并讲
- 拷贝构造函数的用法为`ClassName(const ClassName& c) {}`，该类的每一次拷贝操作都会调用该函数

## 动态数组Vector
- 使用动态数组Vector需要`include<Vector>`，Vector像是Java中的ArrayList，它会随着数据的增加不停的扩容
- Vector的扩容操作是不停的新建一个大数组->进行拷贝->销毁原小数组，可见其中的开销是非常大的，不进行任何优化的话它甚至是每一次添加元素都会来一次，其复制次数为n!,非常辣鸡
- 深拷贝的用法和Vector的优化操作可以合起来看
- Vector的元数据（指向对象数据的指针以及数组长度信息）的存放位置由构建方式决定，而其数组数据一直存放在堆中
```c++
class Entity {
public:
    int X, Y;
public:
    Entity(){

    }
    Entity(const int& x, const int& y) : X(x), Y(y){

    }

    ~Entity(){

    }

    Entity(const Entity& e) : X(e.X), Y(e.Y) {
        //既然重载了拷贝构造函数，那别忘了正常实现拷贝的逻辑功能， 见初始化列表
        std::cout << "copy" << std::endl;
    }

    // ban掉拷贝函数禁止拷贝，unique_ptr就是这么做的
    Entity(const Entity& e) = delete;

};

std::ostream& operator<< (std::ostream& s, const Entity& t) {
    return s << t.X << ',' << t.Y;
}

int main()
{
    // vector的数据总是放在堆里的，但是它的元数据（指向对象数据的指针以及数组长度信息）是根据创建方式来决定是放堆还是栈里
    // 下面的vector的元数据和数组数据都在堆里
    std::vector<Entity>* v = new std::vector<Entity>();
    // 如果事先知道需要的空间可以事先开辟，避免在加元素过程中增长产生消耗
    v->reserve(3);
    // emplace_back需要以参数列表的形式传进去
    v->emplace_back(1, 2);
    v->emplace_back(3, 4);
    v->emplace_back(5, 6);
    // 新加的range-based for语法糖好用
    for (Entity& e : *v) {
        std::cout << e << std::endl;
    }

    // 以下的vector元数据在栈中，数组数据在堆中
    std::vector<Entity> m_v;
    m_v.push_back(Entity(1,2));
    m_v.push_back({ 3,4 });
    m_v.push_back({ 5,6 });

    for (Entity& n : m_v) {
        std::cout << n << std::endl;
    }
}
```
### Vector的优化
- 优化Vector需要以减少拷贝次数入手
  - 事先约定数组的大小，或是事先给个较大的值，可以尽量避免扩容复制的操作
  - 使用emplace_back可以避免push_back中的复制操作
### emplace_back和push_back
- 二者都是添加元素的操作，然而各有不同
- `push_back`: 该操作会实例化一个临时实例，容器拷贝该临时实例放入容器，然后释放该临时实例，由此过程可见存在一个很没必要的拷贝操作
- `emplace_back`: 该操作需要我们传入参数列表，容器接收参数列表后在容器内使用参数完成实例构建并直接放入容器，避免了创建临时实例并拷贝的操作，这也是为什么我们调用时是用圆阔号传列表
- 对于需要高效添加大量数据的场景，emplace_back是一个更好的选择。

## 模板
- 模板像是Java中的泛型，但比泛型要高级的多
- 模板像是一种宏，使用它来操控编译器来生成代码，像是完型填空一样，`需要注意编译器只会创建被用到的函数`，模板在编译阶段执行
```c++
void print(const int& value) {
    std::cout << value << std::endl;
}

void print(const std::string value) {
    std::cout << value << std::endl;
}

int main()
{
    int a = 1;
    std::string s = "name";

    print(a);
    print(s);
}
```
- 对于以上这种情况，是最简单的也是该用模板的情况
```c++
template<typename T>
void print(const T& value) {
    std::cout << value << std::endl;
}
```
- 然而c++的模板要强大的多什么特化什么递归什么，妈的看不懂记不住，下面再抄一个模板用在函数中的操作
```c++
template<int N>
class m_Array {
    int array[N];
public:
    int getsize() {
        return N;
    }
};

int main()
{
    m_Array<5> ma;
    std::cout << ma.getsize() << std::endl;
}
```
- 完型填空既视感，博大精深这里只是皮毛
- `需要注意的是我们写代码以干净整洁为主，所以不要为了使用特性而使用，要以可读性为主（防御性编程除外）`

## auto
- 该关键字一句话就是让编译器自行判断数据类型
- 一般只有在迭代器啥的那种变量名变态长而且逻辑不乱的地方用

## ->箭头运算符
- 用于通过指针调用方法的运算符
- 还可重载用于简化某些代码（见视频）

## 宏
`宏往往用于调试比较多`
- 这些都是宏, 宏定义
```c++
#define PI 3.14159 // 宏变量
#define MAX(a, b) ((a) > (b) ? (a) : (b)) // 宏函数
```
- 用于条件编译
```c++
#define DEBUG

int main() {
    #ifdef DEBUG
    std::cout << "Debug mode is enabled" << std::endl;
    #endif

    #ifndef RELEASE
    std::cout << "Release mode is not defined" << std::endl;
    #endif
}
```
- `虽然宏非常强大，但它们有局限性，比如缺乏类型安全和调试困难。现代 C++ 中通常推荐使用内联函数和模板作为宏的替代方案。`

## STL静态数组（c++风格数组）
- 需要`#include<array>`
- `std:array<int, 5> data = {1,2,3,4,5}` 或 `std:array<int, 5> data{{1,2,3,4,5}}`，这就是静态数组的用法了
- 也是个包装作用，`提供了方法，运算符重载，边界检查，提供迭代器操作，可使用排序操作，而且开销基本和普通c的数组一致，所以一般用这个std:array比较多`
- std:array的数组长度之类的数据通过模板存入读取的（见模板中用于函数的模板例子），所以没有专门存放数组长度的空间开销
- `重点tip：std::array 不支持在堆上创建， 不支持在堆上， 不支持在堆上`

## 函数指针和lambda函数
### c风格版本函数指针（原始函数指针）
- 函数指针顾名思义是指向函数的指针，我们可以将函数赋给一个变量，并将该函数以变量的形式作为参数传递给其他的函数从而实现复杂的逻辑，功能等
- 代码中的函数在编译时会被编译成一个个cpu指令，并存放在二进制文件中，调用时就call那个位置的指令，函数指针就保存了函数所在的位置，当调用时便跳转到指针所指向的指令地址然后执行。
```c++
class Entity {
public:
    std::string name;

    Entity(const std::string name) : name(name) {

    }
};

void print(const std::string& s) {
    std::cout << s << std::endl;
}

void funptr_test(Entity e, void(*fuc_p)(const std::string& s)) {
    fuc_p(e.name);
}
int main()
{   
    auto fuc_p = print;
    // 如果自己写函数指针类型真的很别扭，因为变量名m_p在前面那个括号里面
    void (*m_p)(const std::string& s) = print;
    // 下面俩行和上面的代码等价，使用typedef简化了一下
    typedef void (*m_p)(const std::string& s);
    m_p = print;

    Entity e("name");
    funptr_test(e, fuc_p);
}
```
- auto用在这里就很合适，函数指针的定义非常别扭，如上print的指针类型是`void(*fuc_p)(const std::string& s)`，其中`void`是函数的返回值，`fun_p`是自定义的函数指针的变量名字 啥都行，`const std::string& s`是函数的参数列表。这些东西长且别扭，所以用auto在这里挺合适的，不用强记鼠标放上去就有XD。
- `当然现在写函数指针基本不用c的原始方式了，但需要知道这种用法`

### lambda（有点麻烦，后续补充）
- 可以理解为一个临时函数，它用完即弃并且是我们的代码在运行的过程中生成的，不能算是个真正的函数，定义以`[]`开头，`[]`叫捕获方式，具体的可以自己查查看看
- 它是不需要进行函数定义就能写一个函数的方式
- 简单的用法如下所示，写在参数列表里啥的都行
```c++
auto mf = [](Entity e) { std::cout << e.name << std::endl; };
    mf(e);
```

## namespace 命名空间
- 命名空间的出现就是为了区分函数签名相同但隶属于不同命名空间的函数（可能不严谨）
```c++
namespace apple {
    void print() {
        std::cout << "apple" << std::endl;
    }
}

namespace orange {
    void print() {
        std::cout << "orange" << std::endl;
    }
}
int main()
{   
    apple::print();
    orange::print();
}
```
- 如果同时using多个命名空间，若某个函数签名相同则报错，因为有歧义
- 多个命名空间下，最接近的函数为优先调用的函数，如下所示，由于apple中的print不需要隐式转换，所以该函数优先调用
- `using namespace apple`可以省掉双引号之前的命名空间，但建议是不要用，理由如下
  -  读者需要找该定义的位置来确定方法的命名空间所属
  -  `而且永远永远不要在头文件中用，它会导致bug很难很难追踪`
- 但在小作用域中可以小用，但一定要控制好
```c++
namespace apple {
    void print(const char s[]) {
        std::cout << s << "apple" << std::endl;
    }
}

namespace orange {
    void print(const std::string s) {
        std::cout << s << "orange" << std::endl;
    }
}
int main()
{
    // 输出i'am apple
    using namespace apple;
    using namespace orange;
    print("i'm ");

    // 该方法可以将orange起别名为a，常用于命名空间嵌套时简化用
    namespace a = orange;
    a::print("i'm ");
}
```

## 线程
- `这里只是简单的实例下线程`
- `#include <thread>` 标准线程库
- `#include <chrono>` 计时库，还有其他的基于平台的性能更吊的计时库，不过这个够用
```c++
#include <chrono>
#include <thread>

// 这个是简化用那个1s中的s的namespace
using namespace std::literals::chrono_literals;

// 全局变量控制线程终止条件
static bool flag = true;

void work() {
    while (flag) {
        // \n这个东西比std::endl要快
        std::cout << "working...\n";
        std::this_thread::sleep_for(1s);
    }
}

int main()
{
    // c++最简单的开线程就是下面这样，把函数传进去，然后拿个线程的变量做其他操作
    auto m_thread =  std::thread(work);
    std::cin.get();
    flag = false;

    // 下面这个join和其他Java，c#中的wait是同一个意思，就是指等待该线程执行完毕后再继续执行当前线程（这里指的是主线程，因为在main里）
    m_thread.join();
    
    std::cout << "over\n";
    std::cin.get();
}
```

## 计时（没想到吧，计时竟然会非常重要）
- 计时功能在基准测试中非常重要，因为选择c++就是选择性能，时间优化极其重要
- `#include <chrono>`别忘了计时库
- `auto start = std::chrono::high_resolution_clock::now();` 获取当前时间戳
- `std::chrono::duration<float> duration = end - start;` 计算俩时间戳的差统计时间
```c++
using namespace std::literals::chrono_literals;

static bool flag = true;

void work() {
    auto start = std::chrono::high_resolution_clock::now();
    while (flag) {
        std::cout << "working...\n";
        std::this_thread::sleep_for(1s);
    }

    auto end = std::chrono::high_resolution_clock::now();

    std::chrono::duration<float> duration = end - start;

    std::cout << duration.count() << std::endl;
}

int main()
{
    auto m_thread =  std::thread(work);
    std::cin.get();
    flag = false;

    m_thread.join();
    
    std::cout << "over\n";
    std::cin.get();
}
```

- 当然这样写法太麻烦了，我们可以借用c++对象的生命周期来帮助我们
- 我们在类的构造函数中保存start，并在析构时保存end，在我们需要计时的作用域中创建一个栈对象就行，当作用域结束自然而然的调用析构函数并完成计时操作
- 这种方式叫`插码(instrumentation)`，可以使用这种插码的形式修改源代码从而包含各种工具
```c++
// 这个是简化用那个1s中的s的namespace
using namespace std::literals::chrono_literals;

static bool flag = true;

class Time {
    std::chrono::steady_clock::time_point start;
    std::chrono::steady_clock::time_point end;
public:
    Time() {
        start = std::chrono::high_resolution_clock::now();
    }

    ~Time() {
        end = std::chrono::high_resolution_clock::now();

        std::chrono::duration<float> duration = end - start;
        std::cout << duration.count() << std::endl;
    }
};

void work() {

    Time t;

    while (flag) {
        std::cout << "working...\n";
        std::this_thread::sleep_for(1s);
    }
}

int main()
{
    auto m_thread =  std::thread(work);
    std::cin.get();
    flag = false;

    m_thread.join();
    
    std::cout << "over\n";
    std::cin.get();
}
```

## 排序
- c++的算法库提供了很多算法实现，这里以排序算法为例
- 别忘了`#include <algorithm>`
- 用法为`std::sort(container.begin(), container.end(), std::greater<int>());`，前两个参数为容器类的迭代器，最后一个是排序规则函数，用标准库的和自定义的都行
```c++
#include <iostream>
#include <algorithm>
#include <vector>
#include <array>

struct Vector {
    int X, Y;
    Vector(int x, int y) : X(x), Y(y){}
};

class Entity {
public:
    Vector v;
    std::string name;

    Entity(Vector v, const std::string& name) : v(v), name(name){

    }
};

std::ostream& operator<< (std::ostream& s, const Entity& e) {
    s << e.v.X << "," << e.v.Y << "\n";
    return s;
}

int main()
{
    Entity e1(Vector(1, 2), "e1");
    Entity e2(Vector(3, 4), "e2");
    Entity e3(Vector(5, 6), "e3");

    std::vector<Entity>* entityArray = new std::vector<Entity>();
    entityArray->reserve(3);
    entityArray->emplace_back(e2);
    entityArray->emplace_back(e1);
    entityArray->emplace_back(e3);

    std::array<int, 3> a{ {3,1,2} };
    // 借用标准库中提供的简单函数方法进行排序
    std::sort(a.begin(), a.end(), std::greater<int>());

    // 自定义排序函数，第三个参数用lambda，函数指针啥都行
    std::sort(entityArray->begin(), entityArray->end(), [](Entity a, Entity b) {
        return a.v.X > b.v.X;
        });

    for (Entity e : *entityArray) {
        std::cout << e << "\n";
    }
}
```
- 对于基础数据类型， 标准库里的排序函数就够用了，对于自定义的类型还是自己写排序函数现实点

## c++的类型双关
- 类型双关最最最核心的理论就是：`c++中类型没有意义，意义在于分配的内存大小`
- 内存双关就是我们对某个类型的内存进行另一种类型的解释，把它解释为任意数据类型，类型双关实际上就是C语言中的指针类型转换操作,利用不同类型的指针对内存进行操作
- 像是以下操作，我们通过int指针对double数据进行操作，就是玩
```c++
int main()
{
	double a = 1.1;
	int* b = (int*) & a;

	b[1] = 1;

	std::cout << a;
}
```

## c++的联合体
- `联合体不支持继承，虚函数等多态操作`
- `通常匿名使用Union，匿名union不能含有成员函数。因为匿名联合体设计的初衷是用来共享数据的，不是用来逻辑处理啥的用的`
- 和c里的联合体一样的，所有成员占同一块内存，从而联合体的空间由最大变量的大小决定
```c++
union MyUnion
{
	int a;
	double b;
};
```
- 总是疑惑联合体这个特性有啥用，但联立类型双关的概念来看它总是有着高级用法
- 看以下例子，Vector4由两个Vector2组成，所以可以借用Vector2来快速的读取Vector4从而不用再刻意为Vector4写一套方法,`这里我们使用c++的类型双关来实现这样的功能`
```c++
struct Vector2
{
	int x, y;

	Vector2(int x, int y) : x(x), y(y){

	}
};

struct Vector4
{
	int a, b, c, d;

	Vector4(int a, int b, int c, int d) : a(a), b(b), c(c), d(d){

	}

	Vector2 getAPart() {
		
		return  *(Vector2*)&(this->a);
	}

	Vector2 getBPart() {

		return *(Vector2*)&(this->c);
	}
};

std::ostream& operator<< (std::ostream& s, Vector2 v) {
	s << v.x << "," << v.y << "\n";
	return s;
}

int main()
{
	Vector2 v2(1,2);

	Vector4 v4(1, 2, 3, 4);

	std::cout << v4.getAPart() << v4.getBPart();
}
```
- 借用联合体我们可以更巧妙的实现以上的功能
- `联合体和结构体不写名称为匿名联合体和匿名结构体，此时它们仅仅是一种数据结构，不带有其他特性`
- 这样x和a，b共享内存，y和c，d共享内存
```c++
struct Vector2
{
	int x, y;
};

struct Vector4 {

	union
	{
		struct
		{
			int a, b, c, d;
		};

		struct
		{
			Vector2 x, y;
		};
	};
};

std::ostream& operator<< (std::ostream& s, Vector2 v) {
	s << v.x << "," << v.y << "\n";
	return s;
}

int main()
{
	Vector4 v{ 1,2,3,4 };

	std::cout << v.x << v.y;
}
```
-`针对以上代码：`以上代码里的Vector4好像没办法通过两个Vector2进行初始化，`但如果将Vector4中联合体里的两个结构体调换顺序`就可以使用两个Vector2进行初始化了，为啥
```c++
struct Vector4 {

	union
	{
		struct
		{
			Vector2 x, y;
		};

		struct
		{
			int a, b, c, d;
		};
	};
};

int main()
{
	Vector4 v{ 1,2,3,4 };

	Vector2 v1{ 1,2 }, v2{ 3,4 };

	Vector4 mv{v1, v2};

	std::cout << mv.x << mv.y;
}
```

## 虚析构函数
- 虚析构函数和虚函数设计的目的一致
- 如果父类`没有将析构函数指定为虚函数`，并且子类在继承父类后添加了成员变量等一些需要进行手动销毁的东西时，此时销毁子类仅仅会调用父类的析构函数，从而子类里的那些资源会造成内存泄漏
- 当父类`指定析构函数为虚函数后`，销毁子类时会依次调用子类，父类的析构函数，从而将内存一层层完全释放
- `务必记得将父类的析构函数指定为虚函数`

## 预编译头文件
- 每个c文件中的头文件之前已经知道都是复制粘贴的操作写在里面的，如果每个c文件中我们都加一大串的头文件，这意味着每个文件都要对这些一大串的头文件进行编译，会造成大量重复编译的问题
- 我们通过将一些永远不会动的，广泛使用的头文件设置为预编译，此时它们便只会被编译一次，大大的减少程序的编译时间

## c++类型转换
- 原始c语言的类型转换：`int(a), (int)a`，原始的类型转换目前不推荐使用
  - 有些场景比如：想要搜索代码中进行类型转换的部分时，原始的类型转换无法支持我们进行搜索工作
  - c++的类型转换添加了许多检查，虽然会降低性能，但降低的有限
- c++的类型转换：`static_cast`, `dynamic_cast`, `const_cast`, `reinterpret_cast`
- c++的类型转换都是在传统c的类型转换上包一层东西，如转换是否合法检查等，换句话说c++的类型转换完全可以使用c来完成
### static_cast
- `static_cast<targettype>(usertype)`
- 不存在类型转换安全性检查
### dynamic_cast（较为重点）
- `dynamic_cast<targettype>(usertype)`
- 存在类型转换安全性检查
- `dynamic_cast只能用于多态类类型(也就是某些类型的基类维护一个虚函数表)`
- `dynamic_cast类似于函数，与static_cast在编译阶段进行不同的是dynamic_cast在运行时进行`
- 安全性检查的意义如下所示，一个Entity对象可以表示Player和Enemy，并且二者可以很简单的转为Entity，但将转为Entity的Player和Enemy对象再转回去时就需要类型检查，否则子类产生混淆会非常危险，而dynamic_cast主要就是干这个的
- 它的原理在于每个类会维护一个运行时类型信息(run-time-type-information/RTTI)，从而dynamic_cast根据RTTI进行类型验证
- 自然也会注意到dynamic_cast的性能成本所在：`一个是RTTI`，`一个是验证过程`，所以如果针对性的优化的话就不用dynamic_cast
```c++
class Entity {
public:
	Entity() {
		
	}

	virtual ~Entity() {

	}
};

class Player : public Entity{

};

class Enemy : public Entity {

};
int main()
{
	Entity* e = new Player();
	// return nullptr
	Enemy* enemy = dynamic_cast<Enemy*>(e);
	// return player ptr
	Player* player = dynamic_cast<Player*>(e);
	
}
```

### reinterpret_cast
- `reinterpret_cast<targettype>(usertype)`
- 类型双关式类型转换，就是一段内存采用另一种命名
### const_cast
- `const_cast<targettype>(usertype)`
- 将常量指针啊引用啊通通转为非常量形式，高级


## 条件与操作断点
- 条件断点：满足某些条件时触发该断点
- 操作断点：经过断点处语句时触发某些操作，如打印某些变量等
- 这两个都是IDE提供的功能，通过右键点击断点那个小红点来给断点添加以上这些东西，在调试时候非常非常有用

## 

# Tips
## 声明与实现分离
- 通过将声明放在 .h 头文件中，而将实现放在 .cpp 文件中，编译器只需要编译一次实现部分，然后在链接阶段将这些定义好的函数与其他源文件中的引用进行连接。这样可以提高编译效率，同时也更有利于代码的维护和管理。

