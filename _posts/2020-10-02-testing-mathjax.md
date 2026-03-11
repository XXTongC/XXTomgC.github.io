---
layout: post
title: 智能指针 SmartPointer
tags: mathjax
math: true
date: 2026-03-11 15:32 +0800
---

# 智能指针 SmartPointer

 智能指针就是帮我们C++程序员管理动态分配的内存的，它会帮助我们自动释放new出来的内存，从而避免内存泄漏！记住添加 memory模块
 
```cpp
#include <memory> //or import <memory>;
```

## unique_ptr

unique_ptr< T >对象类似于指向T类型的指针，是唯一的，换而言之，不能有多个unique_ptr<>对象保存相同的地址   
注:虽然不能复制unique_ptr<>,但是使用std::move()函数可以把一个unique_ptr<>对象储存的的地址移动到另一个unique_ptr<>对象中。执行该操作后，最初的智能指针就再次变成空指针。

```cpp
#include <iostream>
#include <memory>
class Entity
{   
public:
    Entity()
    {
        std::cout << "Created Entity" << std::endl;
    }
    ~Entity()
    {
        std::cout << "Destroyed Entity" << std::endl;
    }
    void test(){}
};
int main()
{
    {

        std::unique_ptr<Entity> entity(new Entity());                   //第一种写法
        //std::unique_ptr<Entity> entity = std::make_unique<Entity>();  //第二种写法,并且更加安全
        //std::unique_ptr<Entity> entity = new Entity();                //error, unique_ptr的构造函数实际上是explicit
        //auto entity{std::make_unique<Entity>()};                      //第三种写法，同时也推荐这样写，将优化的事留给编译器去做，这样还能减少代码编写量的同时不影响代码的可读性 
        //std::unique_ptr<Entity> e0 = entity;                          //error,这是unique_ptr的一种自我防卫机制,本质上它删除了拷贝函数,以防止你需要它时却发现它已经自尽了!
        entity->test();
    }                                                                   //print : Destroyed Entity
}
```
以上,我们可以得出的结论是unique_ptr在内置作用域块的结尾自动释放了   
注：make_unique<>()函数在C++14中引入
## shared_ptr
shared_ptr是一个很有趣的智能指针,它会自动计算有几处地方是使用到了它,当它所有的"拷贝"都自尽时,它本人也就跟着自尽了!(此处我将用count模拟它的计数)
```cpp
class Entity
{   
public:
    Entity()
    {
        std::cout << "Created Entity" << std::endl;
    }
    ~Entity()
    {
        std::cout << "Destroyed Entity" << std::endl;
    }
    void test(){}
};
int main()
{
    {
        std::shared_ptr<Entity> e0;
        {
            std::shared_ptr<Entity> entity(new Entity());   //count=1 print : Created Entity
            int a = 1;
            if (a)
            {
                e0 = entity;                                //count=2
            }                                                                   
        }                                                   //count=1                  
    }                                                       //count=0 print : Destroyed Entity
}
```
在上述代码中我们运用了new语句来给shared_ptr分配空间,但这种做法并不完美
```cpp
std::shared_ptr<Entity> e2 = std::make_shared<Entity>();
```
为什么呢?就如之前所说,shared_ptr会有一个count的计数,它在使用时会为count单独分配空间,如果使用new,那么会被重复分配两次,效率降低,而make_shared<>() 语句可以提高程序的效率,避免两次内存的分配

## weak_ptr
当shared_ptr指向另一个shared_ptr 时,count++. 但是,当weak_ptr指向shared_ptr时,count不会++ !所以它可以用来防止内存泄漏。程序员不能直接访问weak_ptr<>封装的地址，编译器会强制首先从weak_ptr<>创建一个share_ptr<>对象来引用相同的地址。

*******
## 关于智能指针的运用
1.用get()成员函数可以访问智能指针包含的地址(以十六进制形式输出)     
2.指向数组的智能指针：
```cpp
auto padata_3{ std::make_unique<double[]>(3) };         //指padata_3指向包含3个double类型元素的数组
```
注意：在使用std::make_unique< T[] >()去声明数组时数组会将元素初始化为0，这样做是有额外性能消耗的，因为它需要先清空内存再赋值，在C++20中可以使用std::make_unique_default_init< T >()或std::make_unique_defaule_init< T[] >()去创建（std::share_ptr同理），但这样做的结果又导致了元素值的不确定。    
3.通过reset()成员函数可以重置unique_ptr中包含的指针或任意类型的智能指针
```cpp
auto ptr {std::make_unique<double>(3.3)};
ptr.reset(new double {2.2});   //*ptr=2.2
ptr.reset(); //ptr=nullptr
```
4.通过release()成员函数将智能指针转换为原始指针，但要注意的是当调用release()后，程序员需要自己delete或delete[]，因此非必要不要使用release()成员函数。    
警告：当使用release()时千万不要使用以下写法，因为它将会导致你失去ptr的地址，从而导致内存泄漏，所以一定要使用原始指针去接收它。同时需要注意的是，虽然reset()与release()调用时都会将指针指向nullptr，但是reset()会释放智能指针之前拥有的任何内存，而release()不会。
```cpp
ptr.release();              //Dangerous way
double* p=ptr.releasse();   //Right way
```


