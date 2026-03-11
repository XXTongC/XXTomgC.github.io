---
layout: post
title: 享元模式
tags: 设计模式
---
# 享元模式(Flyweight)

享元模式常常会用于有大量相似对象的场景下，我们会考虑如何将这些相似的对象储存起来的开销最小。

通常，我们会将享元模式下的对象分为**可共享的内部状态**和**不可共享的外部状态**。

## 享元模式的典型结构

```txt
┌─────────────────┐
│     Client      │ (客户端)
└────────┬────────┘
         │ 请求享元对象
         │ 传递外部状态
         ↓
┌─────────────────────────┐
│   FlyweightFactory      │ (享元工厂)
├─────────────────────────┤
│- flyweights: Map        │ (享元对象池)
├─────────────────────────┤
│+ getFlyweight(key)      │ (获取或创建享元)
└──────────┬──────────────┘
           │ 创建/管理
           │ 返回共享对象
           ↓
┌──────────────────────┐
│     Flyweight        │ (抽象享元)
│   <<interface>>      │
├──────────────────────┤
│+ operation(extrinsic)│ (接受外部状态)
└──────▲───────────────┘
       │
       │ 实现
       ├─────────────────────────┐
       │                         │
┌──────┴──────────────┐  ┌───────┴────────────┐
│ConcreteFlyweight    │  │UnsharedFlyweight   │
├─────────────────────┤  ├────────────────────┤
│- intrinsicState     │  │- allState          │ (不共享)
├─────────────────────┤  ├────────────────────┤
│+ operation(extrinsic)│  │+ operation(...)    │
└─────────────────────┘  └────────────────────┘
   (共享的享元对象)         (不需要共享的对象)
```

| 角色 | 职责 | 说明 |
|------|------|------|
| **Flyweight** | 抽象享元 | 声明接口，通过该接口享元可以接受并作用于外部状态 |
| **ConcreteFlyweight** | 具体享元 | 实现Flyweight接口，存储内部状态（可共享） |
| **UnsharedFlyweight** | 非共享享元 | 不需要共享的Flyweight子类（可选） |
| **FlyweightFactory** | 享元工厂 | 创建并管理享元对象，确保合理地共享 |
| **Client** | 客户端 | 维护外部状态，调用享元对象时传递外部状态 |

其工作流程大致如下：

```txt
客户端请求 → FlyweightFactory.getFlyweight(key)
                    ↓
            检查享元池中是否存在该key
                    ↓
        ┌───────────┴───────────┐
        ↓                       ↓
    存在：返回现有享元      不存在：创建新享元
        │                       │
        │                   添加到享元池
        │                       │
        └───────────┬───────────┘
                    ↓
        返回享元对象给客户端
                    ↓
    客户端调用 flyweight.operation(extrinsicState)
                    ↓
        享元使用内部状态 + 外部状态完成操作
```

## 适用场景

在游戏引擎中，我们会运用到很多纹理内容例如树木和草地等，我们不可能每次在世界中生成一棵树就为这颗树导入一个相同的纹理。常用的做法是创建一个纹理池，若在池中找不到这个纹理再从储存中导入这个纹理并将其储存在纹理池中。

## 示例

思考我们有一个需求，需要将一个文本的部分小写字母改为大写输出，比较幼稚的做法是为每个字母做一个是否为大小写的映射：

```cpp
std::string targetText;
bool* if_caps;
```

这种做法有什么缺点呢？

1. 浪费大量内存
2. 扩展性差，如果我们还需要将字符进行斜体、加粗等功能呢？

那此时我们又会想到：储存需要更改的范围并将需要更改的相同的属性定义在一个类中。

以下是其完整的实现：

```cpp
class BetterFormattedText
{
public:
    struct TextRange
    {
        int start, end;
        bool capitalize{false};
        // other option here, e.g bold, italic, etc.
        // determine our range covers a particular position
        bool covers(int position) const 
        {
            return position >= start && position <= end;
        }
    }

    TextRange& get_range(int start,int end)
    {
        formatting.emplace_back(TextRange{start, end});
        return *formatting.rbegin();
    }
private:
    std::string plain_text;
    std::vector<TextRange> formatting;
}
```

此时，当我们需要用其享元数据输出我们的文本内容时可以这样做：

```cpp
friend ostream& operator<<(ostream& os, const BetterFormattedText& obj)
{
    std::string s;
    for(size_t i = 0;i < obj.plain_text.length();++i)
    {
        auto c = obj.plain_text[i];
        for(const auto& rng : obj.formatting)
        {
            if(rng.covers(i)&&rng.capitalize)
                c.toupper(c);
            s += c;
        }
    }
    return os << s;
}
```