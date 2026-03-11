---
layout: post
title: 装饰器模式
tags: 设计模式
---
# 装饰器模式(Decorator)

通常情况下需要扩展一个类的时候我们会想到继承：设计一个派生类添加想要的功能，甚至覆写父类的虚函数。

但这并不总是最有效的方法，例如我们很少去继承 std::vector ，因为其缺少虚析构函数。继承不起作用的关键原因是我们需要实现多个强化的功能，并且由于单一职责原则，我们希望将这些功能分开。

此时装饰器(Decorator)模式允许我们在既不修改原始类型（违背开闭原则）也不会产生大量派生类的情况下强化既有类型的职责和功能。

## 装饰器模式的典型结构

```txt
┌─────────────────┐
│   Component     │ (抽象组件接口)
│  <<interface>>  │
├─────────────────┤
│ + operation()   │
└────────▲────────┘
         │
         │ 实现
    ┌────┴────────────────────┐
    │                         │
┌───┴──────────────┐  ┌───────┴────────┐
│ConcreteComponent │  │   Decorator    │ (装饰器基类)
├──────────────────┤  ├────────────────┤
│+ operation()     │  │- component     │ (持有Component引用)
└──────────────────┘  │+ operation()   │
                      └───────▲────────┘
                              │
                              │ 继承
                    ┌─────────┴──────────┐
                    │                    │
            ┌───────┴─────────┐  ┌───────┴─────────┐
            │DecoratorA       │  │DecoratorB       │
            ├─────────────────┤  ├─────────────────┤
            │+ operation()    │  │+ operation()    │
            │+ addedBehavior()│  │+ addedBehavior()│
            └─────────────────┘  └─────────────────┘
```

| 角色 | 职责 | 说明 |
|------|------|------|
| **Component** | 抽象组件 | 定义对象接口，可以给这些对象动态添加职责 |
| **ConcreteComponent** | 具体组件 | 定义具体对象，装饰器可以给它添加额外职责 |
| **Decorator** | 抽象装饰器 | 持有一个Component对象，并实现Component接口 |
| **ConcreteDecorator** | 具体装饰器 | 向组件添加具体职责 |

其工作流程大致如下：

```txt
客户端请求 → ConcreteDecoratorB → ConcreteDecoratorA → ConcreteComponent
                    ↓                      ↓                    ↓
                添加功能B              添加功能A            核心功能
                    ↓                      ↓                    ↓
                    ←──────────────────────←────────────────────
                              返回结果（层层包装）
```

装饰器模式一般有两种实现模式：

* **动态组合** 允许在运行时组合某些东西，通常是通过按引用传递实现的。它的灵活性强，因为组合可以在运行时相应用户的输入。
* **静态组合** 意味着对象及其强化功能是在编译期时使用模板组合而成的。这意味着在编译时需要知道对象确切的强化功能，因为之后无法对其进行修改。

## 动态装饰器

考虑为一个 Shape 类扩展关于颜色的功能。使用组合而不是继承实现 Colored-Shape，传入一个 Shape 的引用，这时候 ColoredShape 就可以在已构造好的 Shape 上强化其功能：

```cpp
struct Shape
{
    virtual std::string str() const = 0;
};

struct Circle : Shape
{
    float radius;
    explicit Circle(const float radius)
	    :radius(radius)
    {}

    void resize(float factor)
    {
        radius *= factor;
    }

    std::string str() const override
    {
        std::ostringstream oss;
        oss << "A circle of radius " << radius;
        return oss.str();
    }
};

struct ColoredShape : Shape
{
    Shape& shape;
    std::string color;

    ColoredShape(Shape& shape,const std::string& color)
	    :shape(shape),color(color)
	{}

    std::string str() const override
    {
        std::ostringstream oss;
        oss << shape.str() << " has the color " << this->color;
        return oss.str();
    }

    void make_dark()
    {
        if (constexpr auto dark = "dark "; !color.starts_with(dark))
            color.insert(0, dark);
    }
};
```

可以发现，ColoredShape 本身也是一个 Shape，同时组合了一个它装饰的 Shape 的引用。其中 constexpr、if 初始化和 运用C++20的 starts_with() 去创建了一个成员函数 make_dark()。

此时可以如下使用这个装饰器：

```cpp
    Circle circle{ 0.5f };
    ColoredShape redCircle{ circle,"red" };
    std::cout << redCircle.str()<<std::endl;
    // A circle of radius 0.5 has the color red
    redCircle.make_dark();
    std::cout << redCircle.str() << std::endl;
    // A circle of radius 0.5 has the color dark red
```

此时我们想要给 Shape 增加一个透明度的强化功能，那么如下：

```cpp
struct TransparentShape : Shape
{
    Shape& shape;
    uint8_t transparency;

    TransparentShape(Shape& shape,const uint8_t transparency)
	    :shape(shape),transparency(transparency)
	{}

    std::string str() const override
    {
        std::ostringstream oss;
        oss << shape.str() << " has " << static_cast<float>(transparency) / 255.f * 100.f << "% transparency";
        return oss.str();
    }
};
```

此时根据我们组合模式的原理，我们可以设计如下同时具有透明度和颜色属性的圆形：

```cpp
 Circle c{ 5 };
 ColoredShape cs{ c,"green" };
 TransparentShape myCircle{ cs,64 };
 std::cout << myCircle.str();
 // A circle of radius 5 has the color green has 25.098% transparency
```

但是如此写是需要修改代码的：

```cpp
TransparentShape t{ColoredShape{Circle{23},"green"},64};
```

这里是编译器是会报错的（当然这取决于编译器），因为我们这里并不能将临时对象传给一个引用，这会造成悬空。

## 静态装饰器

在上面创建的示例中有一个很有趣的地方，可以试试去调用一下 Circle 中的 resize()。

实际上 resize() 并不是 Shape 接口的一部分，我们并不能在装饰器中调用它。但是如果我们想要访问被装饰对象的属性成员和成员函数，我们可以通过模板，使用 mixin 继承的方式也就是类继承自它的模板参数来实现：

```cpp
/* 
template<typename T>
concept ShapeConcept = requires(const T t) {
    { t.str() } -> std::convertible_to<std::string>;
} && std::is_base_of_v<Shape, T>;
*/

template<typename T> 
struct ColoredShape : T
{
    static_assert(std::is_base_of_v<Shape, T>, "Template argument must be a Shape");

	std::string color;

    ColoredShape(const std::string& color)
	    :color(color)
	{}

    std::string str() const override
    {
        std::ostringstream oss;
        oss << T::str() << " has the color " << color;
        return oss.str();
    }

    void make_dark()
    {
        if (constexpr auto dark = "dark"; !color.starts_with(dark))
            color.insert(0, dark);
    }
};

template<typename T>
struct TransparentShape : T
{
    static_assert(std::is_base_of_v<Shape, T>, "Template argument must be a Shape");
    uint8_t transparency;
    TransparentShape(){};
    TransparentShape(const uint8_t transparency)
	    :transparency(transparency)
	{}

    std::string str() const override
    {
        std::ostringstream oss;
        oss << T::str() << " has " << static_cast<float>(transparency) / 255.f * 100.f << "% transparency";
        return oss.str();
    }
};
```

我们通过让类继承其模板参数，使其一一具备了我们需要的模板的成员函数和成员变量，同时可以通过 static_assert 或 concept-require 语句去约束模板参数，来保证 T 继承自 Shape

```cpp
    ColoredShape<TransparentShape<Circle>> circle{ "green" };
    circle.radius = 2;
    circle.transparency = 0.5;
    std::cout << circle.str()<<std::endl;
    // A circle of radius 2 has 0% transparency has the color green
    circle.resize(2);
    std::cout << circle.str();
    // A circle of radius 4 has 0% transparency has the color green
```

但是这样的方法也有缺点，我们并没有充分利用构造函数，使得我们只能初始化最外层的类，不能够一行代码就完成这个类的实现。

为了实现这个功能，我们需要构建两者的转发构造函数。这些构造函数将接受两个参数：第一个是特定于当前模板类的参数，第二个是将转发给基类的通用参数包：

```cpp
template<typename ...Args>
TransparentShape(const uint8_t transparency,Args... args)
    :T(std::forward<Args>(args)...),transparency(transparency)
{}
// same for ColoredShape
```

需要注意的是初始化的顺序，我们需要先为 T 的构造函数转发参数包，并且需要注意构造函数参数的数量必须准确，否则会因为无法匹配而编译失败。

还有一个问题是不能将这些构造函数声明为显式的（explicite），否则当多个装饰器组合起来后将会被C++的复制列表初始化规则困扰。