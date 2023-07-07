# unique_ptr 有什么用？
我们知道，我们new出来的对象，如果没有delete，就会发生内存泄漏，虽然说给每一个new出来的对象配一个delete很简单，但是代码一旦变得复杂，分支变多，还会抛出异常，那么就很可能漏掉delete，造成内存泄漏。

比如说下面的代码：

```C++
void f() {
   int*p = new int{3};
   int error = doSomething(p);
   if (error)
       return;
   
   finalize(p);
   delete p;
}
```
看起来很合理，但是可能存在一些问题，如果doSomeThing抛出了一个错误，直接return了，p是不是delete不掉？那有人就说了，我给每一个分支都加一个delete，不就行了吗？

```C++
void f() {
   int*p = new int{3};
   int error = doSomething(p);
   if (error){
       delete p;
       return;
   }
   
   finalize(p);
   delete p;
}
```
但是这里又有一个问题，有可能doSomething直接抛出一个异常，那么两个delete p都不会执行，就会发生内存泄漏。

所以，让程序员来管理内存，是不太靠谱的行为，我们希望有一个机制，可以死板地，机械地给每一个new配上一个delete，我们想到了之前按照生命周期来分类对象类型的知识点，为什么自动对象不会发生内存泄漏呢？因为自动对象是定义在函数里的，函数执行完后，自动对象的生命周期肯定就到头了，它肯定就会被释放。

那么我们能不能用对象的生命周期，来管理资源呢？我们可以把我们之前的代码封装成一个类，当对象初始化时候，我们也new一个int数组，当对象析构的时候，我们释放数组：

```C++
class MyInt{
    public:
        MyInt(int i){
            p = new int{i};
        }

        ~MyInt(){
            delete p;
        }

    private:
        int* p;
};
```
使用的过程中，我们只需要初始化MyInt对象即可，myint对象生命周期结束后，就会自动释放资源，不需要我们管理，而且在程序结束前，myint的生命周期肯定结束，不会发生内存泄漏，这是编译器来保证的：

```C++
{
    MyInt myint(3);
}
```

接下来，我们就把它的其它函数正常写出来就可以了，只不过原来new的地方用MyInt替代即可：
```C++
class MyInt{
    public:
        MyInt(int i){
            p = new int{i};
        }

        ~MyInt(){
            delete p;
        }
        int* p;
};

 void doSomeThing(int* p){

}

void finalize(int* p){

}

void f() {
   MyInt my(3);
   int error = doSomething(my.p);
   if (error) {
       return;
    }
   finalize(my.p);
}
```
这里的f函数，不管在哪个地方跳出，my.p都不会发生泄漏，因为my是一个栈空间了的数据，编译器会在函数执行完后自动的释放。但是我们不可能每次new都写一个这个类来封装，所以C++提供了一个类叫unique_ptr来做到这一点。

# unique_ptr的用法

比如说，我们有一个Demo类：
```C++
class Demo{
    public:
        Demo() = default;
        ~Demo() = default;
        DoSomeThing();
};
```
也有一个处理它的函数：

```c++
void DealDemo(const Demo& d){

}
```


如果用unique_ptr来管理Demo指针，会很方便：
```C++
std::unique_ptr<Demo> owner(new Demo());

```

我们可以直接通过->调用里面的函数，把ower当作Demo指针来用：

```C++
owner->DoSomeThing();
```

也可以通过*来获得Demo对象所指的内容：
```C++
DealDemo(*owner);
```

我们注意到，之前的析构函数是delete，但是如果我们传入的ptr是一个数组呢？哪里用delete[]？unique_ptr实现了偏特化，你可以传入unique_ptr<T[]>来告诉编译器，你传入的是一个数组，那么unique_ptr就会使用delete[]来析构ptr，同时unique_ptr还给出了operator[]的实现，取到数组的元素。

# 实现：构造与析构

unique_ptr里，我们通过构造函数，传入要包装的对象的指针，等这个变量本身被析构后，ptr也被析构了，保证不会内存泄漏：

```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
        unique_ptr() noexcept: ptr(nullptr){}
        unique_ptr(T* p) noexcept: ptr(p){}
        ~unique_ptr() noexcept{ delete ptr;}
}
```

我们用的时候，可能会这么用，我们是想让unique_ptr来管理new出来的int指针，但是，可能会发生隐式转换，new int(3)可能会被转换为unique_ptr<int>类型的东西，就会发生问题，所以我们要加一个explicit。
```C++
unique_ptr<int> owner = new int(3);
```

```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
        unique_ptr() noexcept: ptr(nullptr){}
        explicit unique_ptr(T* p) noexcept: ptr(p){}
        ~unique_ptr() noexcept{ delete ptr;}
}
```

# 实现：移动构造与移动赋值

上面我们说的unique_ptr有一个问题，如果我们把一个unique_ptr拷贝到另外一个unique_ptr里，那么所维护的指针也被拷贝到另外一个unique_ptr里，那么两个unique_ptr就会同时指向同一个资源这会使得两个unique_ptr释放后，资源被释放了两次，所以我们要禁止拷贝构造和拷贝赋值：

```C++
template <typename T>
class unique_ptr{
    public:
        unique_ptr(const unique_ptr&) = delete;
        unique_ptr& operator=(const unique_ptr&) = delete;
}
```
unique_ptr只支持移动构造和移动赋值，对于移动构造，我们只需要把原来的unique_ptr的ptr赋值给当前的ptr，然后把原来的unique_ptr的ptr赋值为nullptr:

```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
        unique_ptr(unqiue_ptr&& o) noexcept: ptr(std::exchange(o.ptr, nullptr)){

        }
}
```
对于移动赋值函数，我们只需要把现在的unique_ptr的ptr先delete掉，然后接管旧unique_ptr里的ptr的所有权，然后剥夺旧unique_ptr的所有权：

```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
        unique_ptr(unique_ptr&& o) noexcept: ptr(std::exchange(o.ptr, nullptr)){

        }

        unique_ptr& operator=(unique_ptr&& o) noexcep{
            delete ptr;
            ptr = o.ptr;
            o.ptr = nullptr;
            return *this;
        }
}
```

使用过程中，我们可以通过构造和赋值来转移旧的unique_ptr的所有权：

```C++
std::unique_ptr<int> u1(new int(1));
// use u1

std::unique_ptr<int> u2(std::move(u1));

std::unique_ptr<int> u3;
u3 = std::move(u2);
```

使用std::move的原因是，u1/u2是一个左值，而拷贝构造被禁止了，所以我们只能通过std::move把左值转换为右值，触发移动构造。

# 实现：重载运算符

我们需要通过*和->来取到ptr所指向的内容，和里面的函数和变量，也就是说，unique_ptr可以当成普通的指针来用：

```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
       T& operator*() const noexcep{
            return *ptr;
       }

       T& operator->() const noexcep{
            return ptr;
       }
}
```

具体的用法是：
```C++
owner->doSomeThing();
*owner.doSomeThing();
```

# 实现：其它成员函数

unique_ptr是唯一的，但是我们可以解绑，我们直接返回包装的ptr，然后把ptr置空：


```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
       T* release() noexcept{
        T* old = ptr;
        ptr = nullptr;
        return old;
       }
}
```

也可以换一个，但是要删除旧的，因为再析构了，就没办法析构原来的了：
```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
       T* reset(T* p) noexcept{
        delete ptr;
        ptr= p;
       }
}
```

然后是获得ptr，和判定ptr是否为空：
```C++
template <typename T>
class unique_ptr{
    T* ptr;
    public:
       T* get() noexcept{
        return ptr;
       }

       explicit operator bool() const noexcept{
        retunr ptr != nullptr;
       }
}
```
# make_unique

通过观察之前的使用方式，我们注意到，我们仍然需要new出Demo对象，但是没有了delete，因为unique_ptr内部已经帮我们delete了：
```C++
std::unique_ptr<Demo> owner(new Demo());

```
这样很不优雅，我们希望，new/delete一起出现，所以我们希望把new去掉，我们可以用make_unique来创建对象，这样就没有了new：

```C++
std::unique_ptr<Demo> owner(std::make_unique<Demo>());
```
make_unique的大概实现方式是有一个变长的形参列表，在()接收构造Demo对象的参数，然后在里面new出对象出来返回，LevelDB里有一个类似的类，来生成一个内存对齐的对象。