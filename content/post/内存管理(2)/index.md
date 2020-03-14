---
title: '内存管理相关内容(2)'
subtitle: '主要对Jie Hou的一些视频所讲的有关内存管理方面的知识的学习'
summary: 本文主要讨论全局new/delete与成员new/delete的差异，同时编写与改进一个小的内存池，为之后理解分配器源码进行铺垫
authors:
- admin
tags: ["C++", "member new",  "Memory Pool"]
categories: [C++]
date: "2020-03-14T00:00:00Z"
lastmod: "2020-03-14T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
image:
  placement: 1
  caption: ''
  focal_point: "Center"
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

重载全局的`::operator new / ::operator delete`，注意，这个重载是不能够放到任何一个`namespace`中的，而且影响也是非常深远的，所以尽量不要对全局的`::operator new / ::operator delete`进行自以为是的操作（本身的源码见`llvm/libcxx/src/new.cpp`）

```cpp
void* myAlloc(size_t size)
{
    return malloc(size);
}

void myFree(void* ptr)
{
    return free(ptr);
}

inline void* operator new(size_t size)
{
    cout << "global new()" << endl;
    return myAlloc(size);
}

inline void* operator new[](size_t size)
{
    cout << "global new[]()" << endl;
    return myAlloc(size);
}

inline void operator delete(void* ptr)
{
    cout << "global delete()" << endl;
    myFree(ptr);
}
```

更加有用的是在类中进行重载

```cpp
class Foo {
public:
    static void* operator new (size_t);
    static void* operator delete(void*, size_t);//size_t is optional
}

Foo* p = new Foo;
//=> 
try {
    void* mem = operator new(sizeof(Foo));
    p = static_cast<Foo*>(mem);
    p->Foo::Foo(1, 2);
}


delete p;
//=>
p->~Foo();
operator delete(p);
```

其实Foo中两个函数都必须是静态的，但是因为历史原因(？)，我们直接不加`static`也能够正确调用，类似的，`operator new[] / operator delete[]`操作方式也是类似的

接下来进行一些骚操作，重载`placement new / delete`，但是每个版本的声明**必须第一个参数为`size_t`**，其余参数以`new`所指定的placement arguments顺序传入（可以随意设计）：

```cpp
Foo* pf = new(300, 'c')Foo;
```

类似的我们可以重载对象成员`operator delete()`，但是他们不会被`delete`调用！只有当`new`所调用的构造函数抛出异常，才会调用重载的`operator delete()`，来归还错误的构造而申请出来的内存，如果不写配套的`delete`，代表放弃处理对应的构造函数抛出的异常

对于placement new来说有个比较好的例子就是`basic_string`

<hr>

接下来主要要来做一个小型的内存池，也是写分配器的第一步

```cpp
class Screen
{
public:
    Screen(int x) : i(x){};
    int get() { return i; }

    //手动接管类的内存
    void *operator new(size_t);
    void operator delete(void *, size_t);
    // ...

private:
    int i;//内容物

private:
    //自建的分配器第一代
    Screen *next;
    static Screen *freeStore;
    static const int screenChunk;
};

Screen* Screen::freeStore = 0;
const int Screen::screenChunk = 24;

void *Screen::operator new(size_t size)
{
    Screen* p;
    if(!freeStore)
    {
        //当linklist为空时，表示空间不够，申请一大块内存
        size_t chunk = screenChunk * size;
        freeStore = p = reinterpret_cast<Screen*>(new char[chunk]);
        //对一大块内存进行分割，然后以linkedlist的方式串联起来
        for(;p != &freeStore[screenChunk-1]; ++p)
            p->next = p + 1;
        p->next = 0;
    }
    p = freeStore;
    freeStore = freeStore->next;
    return p;
}

void Screen::operator delete(void *p, size_t size)
{
    //将delete掉的obj空间重新归还到free list中来
    (static_cast<Screen*>(p))->next = freeStore;
    freeStore = static_cast<Screen*>(p);
}
```

这里引发一个问题，我们需要对每个对象额外增加一个`next`的指针，这对于每个对象来说增加了4个字节，而我们得到的好处是不再需要cookie，一次申请可以之后就可以交给我们来进行手动分配，对于这个解决方案还是不好，一旦这个obj有成千上万个（比如游戏中的粒子）那么这样的指针开销也不太行

```cpp
int main()
{
    cout << sizeof(Screen) << endl;

    size_t const N = 100;
    Screen* p[N];

    for(int i = 0; i < N; ++i)
        p[i] = new Screen(i);

    for(int i = 0; i < 10; ++i)
        cout << p[i] << endl;

    for(int i = 0; i < N; ++i)
        delete p[i];
}

//Output:
16
0x7fbeffc017a0
0x7fbeffc017b0
0x7fbeffc017c0
0x7fbeffc017d0
0x7fbeffc017e0
0x7fbeffc017f0
0x7fbeffc01800
0x7fbeffc01810
0x7fbeffc01820
0x7fbeffc01830
```

因为本机在64位下，指针的大小为8字节，int大小为4字节，内存对齐之后为16个字节，为了证明，也可以在把`int`改成`int64_t`（或者额外增加一个`int`），结果依然还是16字节大小

```cpp
    Screen *next;//8
//class中内容与内存间隔的关系
//全局new与delete情况下
    int i;//=16 space = 16
    int j;//=16 space = 16
    int k;//=24 space = 32
    int l;//=24 space = 32
    int m;//=32 space = 32
    int n;//=32 space = 32

//我们自己写的member new / delete
    int i;//=16 space = 16
    int j;//=16 space = 16
    int k;//=24 space = 24 //这两行凸显了区别！
    int l;//=24 space = 24
    int m;//=32 space = 32
    int n;//=32 space = 32
```

从自己的电脑上得到的结果来看，好像并不是抛弃了cookie，而是手动节约了内存（全局申请出来的都是16字节的倍数，而我们给改成8的倍数），这里与Jie Hou所讲不同（也许是编译器进行了优化）

<hr>

现在我们对刚刚的分配器进行改进，使用`embedded pointer`的方式来优化原来对于每个对象都需要一个额外指针的问题

```cpp
class Airplane
{
private:
    struct AirplaneRep {
        unsigned long miles;//8
        char type;//1
        char test[8];//8
    };

private:
    //将前8个字节看成一个指针，embedded pointer
    //借用前8个字节当做一个指针来用，基本上所有的内存管理都用了这种技巧
    union {
        AirplaneRep rep;
        Airplane* next;
    };

public:
    unsigned long getMiles() { return rep.miles; }
    char getType() { return rep.type; }
    void set(unsigned long m, char t)
    {
        rep.miles = m;
        rep.type = t;
    }

public:
    static void* operator new(size_t size);
    static void operator delete(void* deadObject, size_t size);

private:
    static const int BLOCK_SIZE;
    static Airplane* headOfFreeList;
};

const int Airplane::BLOCK_SIZE = 512;
Airplane* Airplane::headOfFreeList = nullptr;

void* Airplane::operator new(size_t size)
{
    //当大小出现错误交给全局new处理（错误可能发生于出现继承的时候）
    if(size != sizeof(Airplane))
        return ::operator new(size);

    Airplane *p = headOfFreeList;
    if(p)//如果p有效，就往下移动一个元素
        headOfFreeList = p->next;
    else {
        //这里用掉了头尾两个共8个字节的cookies
        Airplane *newBlock = static_cast<Airplane*>(::operator new(BLOCK_SIZE * sizeof(Airplane)));

        //进行切割，注意要跳过#0，因为它会被传回作为本次的结果
        for(int i = 1; i < BLOCK_SIZE - 1; ++i)
            newBlock[i].next = &newBlock[i++];
        newBlock[BLOCK_SIZE-1].next = 0;
        p = newBlock;
        headOfFreeList = &newBlock[1];
    }
    return p;
}

//operator delete接过一个内存块，如果大小正确就加到free list的前端
void Airplane::operator delete(void *deadObject, size_t size)
{
    if(deadObject == 0) return;
    if(size != sizeof(Airplane))
    {
        ::operator delete(deadObject);
        return;
    }

    Airplane *carcass = static_cast<Airplane *>(deadObject);
    carcass->next = headOfFreeList;
    headOfFreeList = carcass;

int main()
{
    cout << sizeof(Airplane) << endl;
    size_t const N = 100;
    Airplane* p[N];

    for(int i = 0; i < N; ++i)
        p[i] = new Airplane;

    //测试数据是否保持正确
    p[1]->set(1000, 'A');
    p[5]->set(2000, 'B');
    p[9]->set(500000,'C');

    cout << p[1]->getType() << " " << p[1]->getMiles() << endl;
    cout << p[5]->getType() << " " << p[5]->getMiles() << endl;
    cout << p[9]->getType() << " " << p[9]->getMiles() << endl;

    for(int i = 0; i < 10; ++i)
        cout << p[i] << endl;

    for(int i = 0; i < N; ++i)
        delete p[i];
}
}
```

```cpp
/*
Output:
24
A 1000
B 2000
C 500000
0x7f8ee8001600
0x7f8ee8001618
0x7f8ee8001630
0x7f8ee8001648
0x7f8ee8001660
0x7f8ee8001678
0x7f8ee8001690
0x7f8ee80016a8
0x7f8ee80016c0
0x7f8ee80016d8
*/
```

也就是跟上面的情况一样，如果是全局的操作，则会以16的倍数对齐，而我们现在修改得以以8为倍数进行对齐

有一点小疑问（主要我很久不用`union`，好多都忘记了）这里记录一下思路

+ 一旦进行`set`操作，`*next`就已经失效，应为`rep`与`*next`是共享同一块内存的

Q：那么失效之后我怎么交还掉这块内存呢？（我一开始也疑惑了qaq）

+ 仔细回忆整个过程，`next`指针什么时候用到？在分配之前从内存池链表中断开和销毁归还后进行串联回内存池中才用到，而具体使用过程中是不会依靠这个`next`去找下一块内存的情况的。所以这块内存可以两用！在内存池中使用`next`部分来串联起整个内存池，在`new`之后使用它的普通部分进行赋值啦等操作，`delete`后就不用理会之前赋值的所有内容，直接按照指针方式重置串接回内存池
+ 实在不理解就对`p[1]`进行`set`，看前后的`*next`的改变，然后调试`delete`部分即可

但是我们持有很多不使用的块而不归还给OS，这样的行为应该说是我们自作主张，不是个好的行为，之后会讨论与尝试解决这个问题