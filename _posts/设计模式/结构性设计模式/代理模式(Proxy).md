# 代理模式(Proxy)

之前学习装饰器模式的时候，学到了强化功能的不同方法。代理模式也类似，但其目标是正在提供某些内部强化功能的同时准确（或尽可能地）保留正在使用的API。

代理模式没有什么同治的模式，因为不同的人构建的不同类型的代理相当多，并且用于不同目的。这里介绍一些不同的代理对象。

## 智能指针

最典型的STL给予我们的例子就是智能指针。智能指针是一个包装类，它封装了原始指针并维护引用计数，同时还重载了部分运算符。但总的来说智能指针提供了原始指针所具有的接口，所以在原始指针出现的位置，都可以使用智能指针。

## 属性代理

属性代理需要详细地介绍一下，这在游戏引擎中是常用技巧。

在其他语言中我们常常用 “属性” 表示底层成员与该成员的 getter/setter 方法的组合。但是 C++ 中没有内置属性的支持，最常见的就是创建对于该成员的 get/set 方法。但是这同时也意味着当我们要操作 t.foo 时，我们需要分别调用 t.get_foo() 和 t.set_foo(value)。但是当我们想继续使用 t.foo 的方法时，并为其提供特定的访问器 / 修改器时，我们可以构建一个 **属性代理**。

属性代理说白了就是可以根据使用语义伪装为普通成员的类，如下定义：

```cpp
template<typename T>
struct Property
{
    T value;
    Property(const T initial_value)
    {
        *this = initial_value;
    }

    operator T()
    {
        return value;
    }

    T operator=(T new_value)
    {
        return value = new_value;
    }
};
```

如此我们就可以如此使用这个属性代理：

```cpp
struct Creature
{
    Property<int> strength{ 10 };
    Property<int> agility{ 5 };
};

Creature creature;
creature.agility = 20;
auto x = creature.strength;
```

属性代理的一个可能扩展是引入伪强类型，可以使用property<T, int Tag>,以便3使用不同类型定义具有不同作用的值，例如，如果我们希望在相似的类型上支持某种算法，以便可以将两个强度值相加，但强度值和敏捷度不能相加，那么这种方法非常有用。

所以下面我们将介绍一下使用**标签分发(Tag Dispatch)技术**来创建**强类型别名(Strong Type Aliases)**，以避免类型混淆和逻辑错误伪强类型。

### 问题背景：弱类型的困境

```cpp
// ❌ 问题：所有属性都是 int，容易混淆
class Character
{
public:
    int strength;     // 力量
    int agility;      // 敏捷
    int intelligence; // 智力
    int health;       // 生命值
    int mana;         // 魔法值
    
    Character(int str, int agi, int intel, int hp, int mp)
        : strength(str), agility(agi), intelligence(intel)
        , health(hp), mana(mp)
    {}
};

// ❌ 这些操作在语法上都是合法的，但在逻辑上是错误的
void demonstrateProblem()
{
    Character hero(10, 15, 8, 100, 50);
    
    // 问题1：参数顺序错误，编译器无法检测
    Character villain(100, 50, 10, 15, 8);  // 参数顺序错了！
    
    // 问题2：不同语义的值可以相加
    int total = hero.strength + hero.agility;  // 力量+敏捷？这没意义！
    
    // 问题3：可以将力量值赋给生命值
    hero.health = hero.strength;  // 类型相同，但语义不同
    
    // 问题4：函数参数容易传错
    auto improveStats = [](int& str, int& agi) {
        str += 5;
        agi += 3;
    };
    
    // 不小心传反了，编译器不会报错
    improveStats(hero.agility, hero.strength);  // ❌ 参数顺序错了
    
    std::cout << "All operations compiled successfully, but logic is wrong!\n";
}
```

### 解决方法：使用标签创建强类型

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <stdexcept>

// ============ 标签定义 ============
// 用于区分不同语义的类型

struct StrengthTag {};      // 力量标签
struct AgilityTag {};       // 敏捷标签
struct IntelligenceTag {};  // 智力标签
struct HealthTag {};        // 生命值标签
struct ManaTag {};          // 魔法值标签

// ============ 强类型属性模板 ============
// T: 底层类型（如 int, double）
// Tag: 标签类型，用于区分不同的语义

template<typename T, typename Tag>
class StrongProperty
{
private:
    T value;
    std::string name;
    
public:
    // 构造函数
    explicit StrongProperty(const std::string& propName = "unnamed", 
                           const T& initialValue = T())
        : value(initialValue), name(propName)
    {
        std::cout << "StrongProperty<" << typeid(Tag).name() << "> '" 
                  << name << "' = " << value << "\n";
    }
    
    // ✅ 只能从相同标签类型赋值
    StrongProperty& operator=(const StrongProperty<T, Tag>& other)
    {
        value = other.value;
        std::cout << "[ASSIGN] " << name << " = " << value << "\n";
        return *this;
    }
    
    // ❌ 不能从不同标签类型赋值（编译错误）
    template<typename OtherTag>
    StrongProperty& operator=(const StrongProperty<T, OtherTag>& other) = delete;
    
    // ✅ 可以从原始类型赋值
    StrongProperty& operator=(const T& val)
    {
        value = val;
        std::cout << "[SET] " << name << " = " << value << "\n";
        return *this;
    }
    
    // 获取值
    const T& get() const { return value; }
    
    // 类型转换
    operator T() const { return value; }
    
    // ✅ 相同标签类型可以比较
    bool operator==(const StrongProperty<T, Tag>& other) const
    {
        return value == other.value;
    }
    
    bool operator<(const StrongProperty<T, Tag>& other) const
    {
        return value < other.value;
    }
    
    bool operator>(const StrongProperty<T, Tag>& other) const
    {
        return value > other.value;
    }
    
    // ❌ 不同标签类型不能比较（编译错误）
    template<typename OtherTag>
    bool operator==(const StrongProperty<T, OtherTag>& other) const = delete;
    
    template<typename OtherTag>
    bool operator<(const StrongProperty<T, OtherTag>& other) const = delete;
    
    // ✅ 相同标签类型可以相加（返回新的强类型值）
    StrongProperty<T, Tag> operator+(const StrongProperty<T, Tag>& other) const
    {
        StrongProperty<T, Tag> result(name + "+" + other.name, value + other.value);
        std::cout << "[ADD] " << name << "(" << value << ") + " 
                  << other.name << "(" << other.value << ") = " 
                  << result.value << "\n";
        return result;
    }
    
    // ✅ 可以加上原始类型
    StrongProperty<T, Tag> operator+(const T& val) const
    {
        return StrongProperty<T, Tag>(name, value + val);
    }
    
    // ❌ 不同标签类型不能相加（编译错误）
    template<typename OtherTag>
    StrongProperty<T, Tag> operator+(const StrongProperty<T, OtherTag>& other) const = delete;
    
    // 自增/自减
    StrongProperty& operator+=(const StrongProperty<T, Tag>& other)
    {
        value += other.value;
        std::cout << "[+=] " << name << " now = " << value << "\n";
        return *this;
    }
    
    StrongProperty& operator+=(const T& val)
    {
        value += val;
        std::cout << "[+=] " << name << " now = " << value << "\n";
        return *this;
    }
    
    // 输出
    friend std::ostream& operator<<(std::ostream& os, 
                                   const StrongProperty<T, Tag>& prop)
    {
        os << prop.value;
        return os;
    }
};

// ============ 类型别名定义 ============
// 为了方便使用，定义具体的类型别名

using Strength = StrongProperty<int, StrengthTag>;
using Agility = StrongProperty<int, AgilityTag>;
using Intelligence = StrongProperty<int, IntelligenceTag>;
using Health = StrongProperty<int, HealthTag>;
using Mana = StrongProperty<int, ManaTag>;
```

### 使用强类型属性

```cpp
#include "strong_typed_property.h"

class StrongCharacter
{
public:
    Strength strength;
    Agility agility;
    Intelligence intelligence;
    Health health;
    Mana mana;
    
    StrongCharacter(int str, int agi, int intel, int hp, int mp)
        : strength("strength", str)
        , agility("agility", agi)
        , intelligence("intelligence", intel)
        , health("health", hp)
        , mana("mana", mp)
    {}
    
    // ✅ 只接受力量类型的函数
    void increaseStrength(const Strength& bonus)
    {
        strength += bonus;
    }
    
    // ✅ 只接受敏捷类型的函数
    void increaseAgility(const Agility& bonus)
    {
        agility += bonus;
    }
    
    void printStats() const
    {
        std::cout << "\n=== Character Stats ===\n";
        std::cout << "Strength: " << strength << "\n";
        std::cout << "Agility: " << agility << "\n";
        std::cout << "Intelligence: " << intelligence << "\n";
        std::cout << "Health: " << health << "\n";
        std::cout << "Mana: " << mana << "\n";
    }
};

int main()
{
    std::cout << "=== Creating Strong Typed Character ===\n";
    StrongCharacter hero(10, 15, 8, 100, 50);
    
    hero.printStats();
    
    std::cout << "\n=== Valid Operations ===\n";
    
    // ✅ 相同类型可以相加
    Strength bonus1("bonus1", 5);
    Strength bonus2("bonus2", 3);
    Strength totalBonus = bonus1 + bonus2;
    std::cout << "Total strength bonus: " << totalBonus << "\n";
    
    // ✅ 可以增加相同类型的值
    hero.strength += bonus1;
    
    // ✅ 相同类型可以比较
    if (hero.strength > Strength("threshold", 10))
    {
        std::cout << "Hero is strong!\n";
    }
    
    // ✅ 可以赋值相同类型
    Strength newStrength("newStrength", 20);
    hero.strength = newStrength;
    
    std::cout << "\n=== Invalid Operations (Compile Errors) ===\n";
    std::cout << "The following would cause compile errors:\n";
    
    // ❌ 编译错误：不能将力量加到敏捷上
    // auto invalid1 = hero.strength + hero.agility;
    std::cout << "// auto invalid1 = hero.strength + hero.agility;  // ❌ Compile Error\n";
    
    // ❌ 编译错误：不能将力量赋值给生命值
    // hero.health = hero.strength;
    std::cout << "// hero.health = hero.strength;  // ❌ Compile Error\n";
    
    // ❌ 编译错误：不能比较力量和敏捷
    // if (hero.strength > hero.agility) {}
    std::cout << "// if (hero.strength > hero.agility) {}  // ❌ Compile Error\n";
    
    // ❌ 编译错误：函数参数类型不匹配
    // hero.increaseStrength(hero.agility);
    std::cout << "// hero.increaseStrength(hero.agility);  // ❌ Compile Error\n";
    
    // ❌ 编译错误：不能将敏捷加到力量上
    // hero.strength += hero.agility;
    std::cout << "// hero.strength += hero.agility;  // ❌ Compile Error\
    \n";
    
    hero.printStats();
    
    return 0;
}
```

### 支持特定操作的强类型

```cpp
#include <iostream>
#include <string>
#include <type_traits>

// ============ 特性标签 ============
// 用于控制哪些操作是允许的

struct Addable {};      // 可加
struct Subtractable {}; // 可减
struct Multipliable {}; // 可乘
struct Comparable {};   // 可比较

// ============ 高级强类型属性 ============
template<typename T, typename Tag, typename... Traits>
class AdvancedProperty
{
private:
    T value;
    std::string name;
    
    // 检查是否具有某个特性
    template<typename Trait>
    static constexpr bool hasTrait()
    {
        return (std::is_same_v<Trait, Traits> || ...);
    }
    
public:
    explicit AdvancedProperty(const std::string& propName = "unnamed",
                             const T& initialValue = T())
        : value(initialValue), name(propName)
    {}
    
    const T& get() const { return value; }
    operator T() const { return value; }
    
    AdvancedProperty& operator=(const T& val)
    {
        value = val;
        return *this;
    }
    
    // ✅ 只有具有 Addable 特性才能相加
    template<typename U = void>
    std::enable_if_t<hasTrait<Addable>(), AdvancedProperty>
    operator+(const AdvancedProperty& other) const
    {
        std::cout << "[ADD] " << name << "(" << value << ") + " 
                  << other.name << "(" << other.value << ")\n";
        return AdvancedProperty(name + "+" + other.name, value + other.value);
    }
    
    template<typename U = void>
    std::enable_if_t<hasTrait<Addable>(), AdvancedProperty&>
    operator+=(const AdvancedProperty& other)
    {
        value += other.value;
        std::cout << "[+=] " << name << " now = " << value << "\n";
        return *this;
    }
    
    // ✅ 只有具有 Subtractable 特性才能相减
    template<typename U = void>
    std::enable_if_t<hasTrait<Subtractable>(), AdvancedProperty>
    operator-(const AdvancedProperty& other) const
    {
        std::cout << "[SUB] " << name << "(" << value << ") - " 
                  << other.name << "(" << other.value << ")\n";
        return AdvancedProperty(name + "-" + other.name, value - other.value);
    }
    
    template<typename U = void>
    std::enable_if_t<hasTrait<Subtractable>(), AdvancedProperty&>
    operator-=(const AdvancedProperty& other)
    {
        value -= other.value;
        std::cout << "[-=] " << name << " now = " << value << "\n";
        return *this;
    }
    
    // ✅ 只有具有 Multipliable 特性才能相乘
    template<typename U = void>
    std::enable_if_t<hasTrait<Multipliable>(), AdvancedProperty>
    operator*(const T& scalar) const
    {
        std::cout << "[MUL] " << name << "(" << value << ") * " << scalar << "\n";
        return AdvancedProperty(name + "*" + std::to_string(scalar), 
                               value * scalar);
    }
    
    // ✅ 只有具有 Comparable 特性才能比较
    template<typename U = void>
    std::enable_if_t<hasTrait<Comparable>(), bool>
    operator>(const AdvancedProperty& other) const
    {
        return value > other.value;
    }
    
    template<typename U = void>
    std::enable_if_t<hasTrait<Comparable>(), bool>
    operator<(const AdvancedProperty& other) const
    {
        return value < other.value;
    }
    
    template<typename U = void>
    std::enable_if_t<hasTrait<Comparable>(), bool>
    operator==(const AdvancedProperty& other) const
    {
        return value == other.value;
    }
    
    friend std::ostream& operator<<(std::ostream& os, 
                                   const AdvancedProperty& prop)
    {
        os << prop.value;
        return os;
    }
};

// ============ 具体类型定义 ============

// 力量：可加、可比较
using AdvStrength = AdvancedProperty<int, StrengthTag, Addable, Comparable>;

// 敏捷：可加、可比较
using AdvAgility = AdvancedProperty<int, AgilityTag, Addable, Comparable>;

// 伤害：可加、可减、可乘（伤害计算）、可比较
using Damage = AdvancedProperty<int, struct DamageTag, 
                                Addable, Subtractable, Multipliable, Comparable>;

// 防御：可减（减少伤害）、可比较
using Defense = AdvancedProperty<int, struct DefenseTag, 
                                 Subtractable, Comparable>;

// 经验值：只能加、可比较
using Experience = AdvancedProperty<int, struct ExperienceTag, 
                                    Addable, Comparable>;

// ID：只能比较（不能进行算术运算）
using EntityID = AdvancedProperty<int, struct EntityIDTag, Comparable>;
```

使用高级强类型：

```cpp
#include "advanced_strong_property.h"

class GameEntity
{
public:
    EntityID id;
    AdvStrength strength;
    AdvAgility agility;
    Experience exp;
    
    GameEntity(int entityId, int str, int agi, int experience)
        : id("id", entityId)
        , strength("strength", str)
        , agility("agility", agi)
        , exp("exp", experience)
    {}
};

class CombatSystem
{
public:
    // 计算伤害
    static Damage calculateDamage(const AdvStrength& str, int multiplier)
    {
        Damage baseDamage("baseDamage", str.get());
        
        // ✅ Damage 支持乘法
        Damage finalDamage = baseDamage * multiplier;
        
        std::cout << "Final damage: " << finalDamage << "\n";
        return finalDamage;
    }
    
    // 应用防御
    static Damage applyDefense(const Damage& damage, const Defense& defense)
    {
        // ✅ Damage 支持减法
        Damage reducedDamage("reducedDamage", damage.get());
        
        // 注意：这里需要手动计算，因为 Damage 和 Defense 是不同类型
        int finalValue = damage.get() - defense.get();
        if (finalValue < 0) finalValue = 0;
        
        std::cout << "Damage after defense: " << finalValue << "\n";
        return Damage("finalDamage", finalValue);
    }
    
    // 增加经验
    static void gainExperience(Experience& currentExp, const Experience& gained)
    {
        // ✅ Experience 支持加法
        currentExp += gained;
    }
};

int main()
{
    std::cout << "=== Creating Game Entities ===\n";
    GameEntity player(1001, 50, 30, 0);
    GameEntity enemy(2001, 40, 25, 0);
    
    std::cout << "\n=== Valid Operations ===\n";
    
    // ✅ 力量可以相加
    AdvStrength strengthBonus("bonus", 10);
    player.strength += strengthBonus;
    
    // ✅ 力量可以比较
    if (player.strength > enemy.strength)
    {
        std::cout << "Player is stronger!\n";
    }
    
    // ✅ 经验值可以增加
    Experience expGained("expGained", 100);
    CombatSystem::gainExperience(player.exp, expGained);
    
    // ✅ ID 可以比较
    if (player.id == EntityID("playerId", 1001))
    {
        std::cout << "Player ID matched!\n";
    }
    
    // ✅ 伤害计算
    Damage damage = CombatSystem::calculateDamage(player.strength, 2);
    
    // ✅ 应用防御
    Defense enemyDefense("enemyDefense", 30);
    Damage finalDamage = CombatSystem::applyDefense(damage, enemyDefense);
    
    std::cout << "\n=== Invalid Operations (Would Cause Compile Errors) ===\n";
    std::cout << "The following operations are not allowed:\n";
    
    // ❌ 力量不能相减（没有 Subtractable 特性）
    // auto invalid1 = player.strength - enemy.strength;
    std::cout << "// auto invalid1 = player.strength - enemy.strength;  // ❌\n";
    
    // ❌ ID 不能相加（没有 Addable 特性）
    // auto invalid2 = player.id + enemy.id;
    std::cout << "// auto invalid2 = player.id + enemy.id;  // ❌\n";
    
    // ❌ 经验值不能相减（没有 Subtractable 特性）
    // auto invalid3 = player.exp - expGained;
    std::cout << "// auto invalid3 = player.exp - expGained;  // ❌\n";
    
    // ❌ 力量不能相乘（没有 Multipliable 特性）
    // auto invalid4 =  player.strength * 2;
    std::cout << "// auto invalid4 = player.strength * 2;  // ❌\n";
    
    // ❌ 不同类型不能混合运算
    // auto invalid5 = player.strength + player.agility;
    std::cout << "// auto invalid5 = player.strength + player.agility;  // ❌\n";
    
    return 0;
}
```

## 虚拟代理

虚拟代理的核心思想和单例模式中的懒汉模式类似，在需要使用到资源时再创建。这是为了在加载开销大的资源时能够延迟创建它。

```cpp
问题：某些对象创建成本很高
┌─────────────────────────────────────────┐
│ • 加载大文件（图片、视频、文档）         │
│ • 建立数据库连接                         │
│ • 初始化复杂的数据结构                   │
│ • 网络资源获取                           │
└─────────────────────────────────────────┘
                    ↓
解决方案：虚拟代理 = 占位符 + 延迟加载
┌─────────────────────────────────────────┐
│ 1. 创建轻量级代理对象（立即）            │
│ 2. 只在真正需要时才创建真实对象          │
│ 3. 对客户端透明                          │
└─────────────────────────────────────────┘
```

### 示例

假设我们有一个加载图片的需求：

```cpp
class Image
{
public:
    virtual void display() = 0;
    ~Image() = default;
};

class Bitmap : public Image
{
public:
    Bitmap(const std::string& filename)
    {
        std::cout << "loading image from" << filename << "\n";
        // loading image here
    }
    void display() override
    {
        // display image
    }
};
```

那么当我们创建 Bitmap 时将直接触发加载图片的行为：

```cpp
Bitmap img{"picture.png"};
```

但我们想提前创建一个占位，当我们真正需要使用到这张图片的时候再加载它。

此时我们可以创建一个虚拟代理：

```cpp
class LazyBitmap : public image
{
private:
    Bitmap* bmp{nullptr};
    std::string filename;
public:
    LazyBitmap(const std::string& filename)
        :filename(filename)
    {}
    ~LazyBitmap() {delete bmp;}
    void display() override
    {
        if(!bmp)
            bmp = new Bitmap(filename);
        bmp->display();
    }
};
```

如上，我们就能在接口需要某个 image 时传入 LazyBitmap 以实现延迟加载了。

## 通信代理

通信代理很像装饰器模式，但其本质的特点在于装饰器模式操作的是在本地的对象，而通信代理操作的很可能是远程对象，其数据的获取需要通过网络通信：

```txt
┌──────────────┬─────────────────────┬─────────────────────┐
│              │      装饰器         │     通信代理        │
├──────────────┼─────────────────────┼─────────────────────┤
│ 对象位置     │ 本地（同一进程）     │ 远程（不同机器）    │
├──────────────┼─────────────────────┼─────────────────────┤
│ 调用方式     │ 直接方法调用        │ 网络/IPC通信         │
├──────────────┼─────────────────────┼─────────────────────┤
│ 主要目的     │ 增强功能            │ 隐藏远程性           │
├──────────────┼─────────────────────┼─────────────────────┤
│ 客户端感知   │ 知道是装饰          │ 不知道是远程         │
├──────────────┼─────────────────────┼─────────────────────┤
│ 性能开销     │ 几乎无              │ 网络延迟             │
├──────────────┼─────────────────────┼─────────────────────┤
│ 典型例子     │ 日志、缓存、压缩    │ RPC、Web服务         │
└──────────────┴─────────────────────┴─────────────────────┘
```

这里提供一个对比示例：

### 示例

```cpp
#include <iostream>
#include <string>

// ============ 装饰器：本地文件操作 ============
class FileWriter
{
public:
    virtual void write(const std::string& data) 
    {
        std::cout << "写入文件: " << data << "\n";
    }
};

// 装饰器：添加压缩功能
class CompressedFileWriter : public FileWriter
{
private:
    FileWriter* writer;  // 本地对象
public:
    CompressedFileWriter(FileWriter* w) : writer(w) {}
    
    void write(const std::string& data) override
    {
        std::cout << "[装饰] 压缩数据...\n";
        writer->write(data);  // 直接调用，本地操作
    }
};

// ============ 通信代理：远程数据库 ============
class Database
{
public:
    virtual ~Database() = default;
    virtual std::string query(const std::string& sql) = 0;
};

// 真实数据库在远程服务器上
class RemoteDatabase : public Database
{
public:
    std::string query(const std::string& sql) override
    {
        std::cout << "    [数据库服务器] 执行SQL: " << sql << "\n";
        return "查询结果";
    }
};

// 通信代理
class DatabaseProxy : public Database
{
public:
    std::string query(const std::string& sql) override
    {
        std::cout << "  [代理] 连接 db.server.com:3306\n";
        std::cout << "  [代理] 发送SQL查询...\n";
        
        // 实际通过网络调用远程数据库
        RemoteDatabase remoteDB;
        std::string result = remoteDB.query(sql);
        
        std::cout << "  [代理] 接收结果\n";
        return result;
    }
};

int main()
{
    std::cout << "=== 实际应用对比 ===\n\n";
    
    std::cout << "【装饰器】本地文件写入:\n";
    FileWriter writer;
    CompressedFileWriter compWriter(&writer);
    compWriter.write("Hello World");
    std::cout << "→ 文件和程序在同一台机器\n\n";
    
    std::cout << "【通信代理】远程数据库查询:\n";
    DatabaseProxy db;
    std::string result = db.query("SELECT * FROM users");
    std::cout << "结果: " << result << "\n";
    std::cout << "→ 数据库在另一台服务器上\n";
    
    return 0;
}
```

## 值代理

### 2.3.3 设计元素描述

#### 1. 表示层组件

**经济损失统计页面 (LossPage)**
- 职责：展示火灾造成的经济损失统计数据
- 功能：
  - 显示直接经济损失和间接经济损失
  - 按月度、季度、年度统计
  - 损失对比分析
  - 损失趋势图表
  - 数据导出功能
- 交互：调用经济损失服务

**火灾原因分析页面 (ReasonPage)**
- 职责：分析和展示火灾发生的原因统计
- 功能：
  - 显示各类原因的统计数据（存储电池、粉尘、电气等）
  - 原因占比分析
  - 原因趋势分析
  - 饼图和柱状图展示
- 交互：调用火灾原因服务

**趋势预测页面 (TrendPage)**
- 职责：基于历史数据进行趋势预测
- 功能：
  - 经济损失趋势预测
  - 火灾发生频率预测
  - 风险等级预测
  - 预测结果可视化
- 交互：调用趋势预测服务

**报表生成页面 (ReportPage)**
- 职责：生成和导出各类统计报表
- 功能：
  - 选择报表类型（日报、周报、月报、年报）
  - 自定义报表时间范围
  - 报表预览
  - 导出为PDF、Excel、CSV格式
- 交互：调用报表生成服务和数据导出服务