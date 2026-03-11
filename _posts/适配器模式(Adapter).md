# 适配器模式(Adapter)

适配器的概率其实很好理解，它的目的在于允许接口不兼容的对象能够相互协作，经典的实例就是电源适配器，它能够连接不同标准的接口使得它们能够一起工作。

## 核心组成部分

1. 目标接口（Target）：客户端期望的接口
2. 适配者（Adaptee）：需要被适配的类，具有不兼容的接口
3. 适配器（Adapter）：将适配者的接口转换为目标接口的类

## 两种实现方式

1. 类适配器：使用多重继承（在C++中）实现
2. 对象适配器：使用组合关系实现（更常用）

## 适配器模式的适用场景

1. 使用现有类，但其接口与需求不匹配：例如，需要集成第三方库或遗留代码。
2. 创建可重用的类，与不相关或不可预见的类协同工作：适配器可以使原本不兼容的类一起工作。
3. 需要使用多个现有子类，但不可能对每一个都进行子类化以匹配接口：可以创建适配器来适配它们的父类。
4. 需要统一多个类的接口：当系统中存在多个类具有相似功能但接口不同时。

## 示例

### 对象适配器

```cpp
#include <iostream>
#include <string>
#include <memory>

// 目标接口 - 客户端期望的接口
class Target {
public:
    virtual ~Target() = default;
    virtual std::string request() const = 0;
};

// 适配者 - 具有不兼容接口的类
class Adaptee {
public:
    std::string specificRequest() const {
        return "Specific request from Adaptee";
    }
};

// 适配器 - 将适配者接口转换为目标接口
class Adapter : public Target {
private:
    std::shared_ptr<Adaptee> adaptee;

public:
    Adapter(std::shared_ptr<Adaptee> adaptee) : adaptee(adaptee) {}

    std::string request() const override {
        // 转换调用
        std::string result = adaptee->specificRequest();
        return "Adapter: (TRANSLATED) " + result;
    }
};

// 客户端代码
void clientCode(const Target& target) {
    std::cout << target.request() << std::endl;
}

// 具体目标实现 - 为了示例完整性
class ConcreteTarget : public Target {
public:
    std::string request() const override {
        return "ConcreteTarget: Standard request";
    }
};
int main() {
    // 使用原生目标对象
    std::cout << "Client: I can work with Target objects:" << std::endl;
    auto target = std::make_unique<ConcreteTarget>();
    clientCode(*target);
    std::cout << std::endl;

    // 使用适配者对象 + 适配器
    std::cout << "Client: I can work with Adaptee objects through the Adapter:" << std::endl;
    auto adaptee = std::make_shared<Adaptee>();
    auto adapter = std::make_unique<Adapter>(adaptee);
    clientCode(*adapter);

    return 0;
}

```

### 类适配器

```cpp
#include <iostream>
#include <string>

// 目标接口
class Target {
public:
    virtual ~Target() = default;
    virtual std::string request() const = 0;
};

// 适配者
class Adaptee {
public:
    std::string specificRequest() const {
        return "Specific request from Adaptee";
    }
};

// 类适配器 - 使用多重继承
class ClassAdapter : public Target, private Adaptee {
public:
    std::string request() const override {
        // 直接调用父类方法
        return "ClassAdapter: (TRANSLATED) " + specificRequest();
    }
};

// 客户端代码
void clientCode(const Target& target) {
    std::cout << target.request() << std::endl;
}

int main() {
    std::cout << "Client: Using ClassAdapter:" << std::endl;
    ClassAdapter adapter;
    clientCode(adapter);
    
    return 0;
}
```