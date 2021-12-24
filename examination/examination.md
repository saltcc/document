# 附-专题

## 一、shared_ptr的那些坑

### 1.共享所有权

一个动态分配的对象可以在多个shared_ptr之间共享，这是因为shared_ptr支持 copy 操作：

```cpp
shared_ptr<string> ptr1{ new string("hello") };
auto ptr2 = ptr1;　// copy constructor
```

### 2.原理介绍

`shared_ptr`内部包含两个指针，一个指向对象，另一个指向控制块(control block)，控制块中包含一个引用计数和其它一些数据。由于这个控制块需要在多个`shared_ptr`之间共享，所以它也是存在于 heap 中的。`shared_ptr`对象本身是线程安全的，也就是说`shared_ptr`的引用计数增加和减少的操作都是原子的。

![img](picture/shared_ptr_ref.png)

### 3.shared_ptr的风险

你大概觉得使用智能指针就再也高枕无忧了，不再为内存泄露烦恼了。然而梦想总是美好的，使用`shared_ptr`时，不可避免地会遇到循环引用的情况，这样容易导致内存泄露。循环引用就像下图所示，通过`shared_ptr`创建的两个对象，同时它们的内部均包含`shared_ptr`指向对方。

![img](picture/shared_ptr_cross.png)

```cpp
// 一段内存泄露的代码
struct Son;

struct Father　{
    shared_ptr<Son> son_;
};

struct Son {
    shared_ptr<Father> father_;
};

int main() 
{
    auto father = make_shared<Father>();
    auto son = make_shared<Son>();
    father->son_ = son;
    son->father_ = father;
  
    return 0;
}
```

分析一下`main`函数是如何退出的，一切就都明了：

- `main`函数退出之前，`Father`和`Son`对象的引用计数都是`2`。
- `son`指针销毁，这时`Son`对象的引用计数是`1`。
- `father`指针销毁，这时`Father`对象的引用计数是`1`。
- 由于`Father`对象和`Son`对象的引用计数都是`1`，这两个对象都不会被销毁，从而发生内存泄露。

为避免循环引用导致的内存泄露，就需要使用`weak_ptr`，`weak_ptr`并不拥有其指向的对象，也就是说，让`weak_ptr`指向`shared_ptr`所指向对象，对象的引用计数并不会增加：

```cpp
auto ptr = make_shared<string>("senlin");
weak_ptr<string> wp1{ ptr };
cout << "use count: " << ptr.use_count() << endl;// use count: 1
```

使用`weak_ptr`就能解决前面提到的循环引用的问题，方法很简单，只要让`Son`或者`Father`包含的`shared_ptr`改成`weak_ptr`就可以了。

```cpp
// 修复内存泄露的问题
struct Son;

struct Father　{
    shared_ptr<Son> son_;
};

struct Son {
    weak_ptr<Father> father_;
};

int main() 
{
    auto father = make_shared<Father>();
    auto son = make_shared<Son>();
    father->son_ = son;
    son->father_ = father;
  
    return 0;
}
```

同样，分析一下`main`函数退出时发生了什么：

- `main`函数退出前，`Son`对象的引用计数是`2`，而`Father`的引用计数是`1`。
- `son`指针销毁，`Son`对象的引用计数变成`1`。
- `father`指针销毁，`Father`对象的引用计数变成`0`，导致`Father`对象析构，`Father`对象的析构会导致它包含的`son_`指针被销毁，这时`Son`对象的引用计数变成`0`，所以`Son`对象也会被析构。

然而，`weak_ptr`并不是完美的，因为`weak_ptr`不持有对象，所以不能通过`weak_ptr`去访问对象的成员，例如：

```cpp
struct Square {
	int size = 0;
};

auto sp = make_shared<Square>();
weak_ptr<Square> wp{ sp };
cout << wp->size << endl;   // compile-time ERROR
```

你可能猜到了，既然`shared_ptr`可以访问对象成员，那么是否可以通过`weak_ptr`去构造`shared_ptr`呢？事实就是这样，实际上`weak_ptr`只是作为一个转换的桥梁（proxy），通过`weak_ptr`得到`shared_ptr`，有两种方式：

- 调用`weak_ptr`的`lock()`方法，要是对象已被析构，那么`lock()`返回一个空的`shared_ptr`。
- 将`weak_ptr`传递给`shared_ptr`的构造函数，要是对象已被析构，则抛出`std::exception`异常。

既然`weak_ptr`不持有对象，也就是说`weak_ptr`指向的对象可能析构了，但`weak_ptr`却不知道。所以需要判断`weak_ptr`指向的对象是否还存在，有两种方式：

- `weak_ptr`的`use_count()`方法，判断引用计数是否为`0`。
- 调用`weak_ptr`的`expired()`方法，若对象已经被析构，则`expired()`将返回`true`。

转换过后，就可以通过`shared_ptr`去访问对象了：

```cpp
auto sp = make_shared<Square>();
weak_ptr<Square> wp{ sp };
    
if (!wp.expired()) 
{
    auto ptr = wp.lock();      // get shared_ptr
    cout << ptr->size << endl;
}
```

### 4.销毁操作

为`shared_ptr`指定 deleter 的做法很简单：

```cpp
shared_ptr<string> ptr( new string("hello"), 
                        []( string *p ) {
                            cout << "delete hello" << endl;
                        });
ptr = nullptr;
cout << "Before the end of main()" << endl;
/**
delete hello
Before the end of main()
**/
```

与`unique_ptr`不同，标准库并不提供`shared_ptr<T[]>`，因此，使用`shared_ptr`处理数组时需要显示指定删除行为，例如：

```cpp
shared_ptr<string> ptr1( new string[10], 
                         []( string *p ) {
                             delete[] p;
                         });
shared_ptr<string> ptr2( new string[10],
                         std::default_delete<string[]>() );
```

由于不存在`shared_ptr<T[]>`，我们无法使用`[]`来访问数组中的元素，实际上无法访问到数组中的元素。也就是说使用`shared_ptr`来指向数组意义并不大。若想要数组在多个`shared_ptr`之间共享，可以考虑使用`shared_ptr<vector>`或`shared_ptr<array>`。

### 5.更多陷阱

使用`shared_ptr`时，注意不能直接通过同一个 raw pointer 指针来构造多个`shared_ptr`：

```cpp
int *p = new int{10};
shared_ptr<int> ptr1{ p };
shared_ptr<int> ptr2{ p };         // ERROR
```

很明显，每次通过 raw pointer 来构造`shared_ptr`时候就会分配一个控制块，这时存在两个控制块，也就是说存在两个引用计数。这显然是错误的，因为当这两个`shared_ptr`被销毁时，对象将会被`delete`两次。

考虑到`this`也是 raw pointer，所以一不小心就会用同一个`this`去构造多个`shared_ptr`了，就像这样：

```cpp
class Student
{
public:
    Student( const string &name ) : name_( name ) { }

    void addToGroup( vector<shared_ptr<Student>> &group ) {
        group.push_back( shared_ptr<Student>(this) );          // ERROR
    }
private:
    string name_;
};
```

每次调用`addToGroup()`时都会创建一个控制块，所以这个对象会对应多个引用计数，最终这个对象就会被`delete`多次，导致运行出错。解决这个问题很简单，只要让`std::enable_shared_from_this<Student>`作为`Student`的基类：

```cpp
class Student : public std::enable_shared_from_this<Student>
{
public:
    Student( const string &name ) : name_( name ) { }

    void addToGroup( vector<shared_ptr<Student>> &group ) {
        group.push_back( shared_from_this() );              // OK
    }
private:
    string name_;
};
```

需要注意，调用`shared_from_this()`之前需要确保对象被`share_ptr`所持有，理解这一点是很容易的，因为只有当对象被`shared_ptr`所持有，使用`shared_from_this()`所返回的`shared_ptr`才会指向同一个对象。

```cpp
vector<shared_ptr<Student>> group;
// Good: ensure object be owned by shared_ptr
auto goodStudent = make_shared<Student>( "senlin" );
goodStudent->addToGroup( group );
Student badStudent1( "bad1" );
badStudent1.addToGroup( group );           // ERROR
auto badStudent2 = new Student( "bad2" );
badStudent2->addToGroup( group );          // ERROR
```

在调用`shared_from_this()`之前需要确保对象被`shared_ptr`所持有，要使得对象被`shared_ptr`所持有，对象首先要初始化（调用构造函数），所以一定不能在`Student`的构造函数中调用`shared_from_this()`：

```cpp
class Student : public std::enable_shared_from_this<Student>
{
public:
    Student( const string &name, vector<shared_ptr<Student>> &group )
        : name_( name ) 
    {
        // ERROR: shared_from_this() can't be call in object's constructor
        group.push_back( shared_from_this() );
    }
private:
    string name_;
};
```

好了，那么问题来了：

- 要怎样才能防止用户以不正确的方式来创建`Student`对象呢？
- 同时也要使得，我们可以使用不同的构造方式来创建`Student`对象。

可以将`Student`的构造函数声明为`private`的，因此，用户无法直接创建`Student`对象。另一方面，增加`create()`成员函数，在这个函数里面，我们使用 C++11 的 variadic templates 特性，调用`Student`的构造函数来构造对象：

```cpp
class Student : public std::enable_shared_from_this<Student>
{
private:
    Student( const string &name ) : name_( name ) { }

    Student( const Student &rhs ) : name_( rhs.name_ ) { }

    // can have other constructor
public:
    template <typename... Args>
    static shared_ptr<Student> create( Args&&... args ) {
        return shared_ptr<Student>(new Student(std::forward<Args>(args)...));
    }

    void addToGroup( vector<shared_ptr<Student>> &group ) {
        group.push_back( shared_from_this() );
    }
private:
    string name_;
};
```

通过`create()`保证了用户创建的对象总是被`shared_ptr`所持有，可以将`create()`想象成`Student`构造函数的别名：

```cpp
vector<shared_ptr<Student>> group;
auto student1 = Student::create("senlin");
student1->addToGroup(group);
cout << "student1.use_count() = " << student1.use_count() << endl;
// student1.use_count() = 2
```

### 6.性能考虑

```cpp
shared_ptr<std::string> ptr{ new string("hello") };
```

使用这种方式创建`shared_ptr`时，需要执行两次`new`操作，一次在 heap 上为`string("hello")`分配内存，另一次在 heap 上为控制块分配内存。使用`make_shared`来创建`shared_ptr`会高效，因为`make_shared`仅使用`new`操作一次，它的做法是在 heap 上分配一块连续的内存用来容纳`string("hello")`和控制块。同样，当`shared_ptr`的被析构时，也只需一次`delete`操作。

```cpp
auto ptr = std::make_shared<std::string>("hello");
```

