# 单例模式(Singleton)

单例模式因为在测试方面存在一些问题，所以并没有很广泛地运用。此文更多介绍其可能会出现的问题，对于其实现方式这里只贴出相关代码。

```cpp
class SingletonClass
{
public:
    static SingletonClass get_Instance()
    {
        static SingletonClass instance;
        return instance;
    }
    SingletonClass(singletonClass&) = delete;
    SingletonClass(singletonClass&&) = delete;
    SingletonClass operator=(singletonClass&)= delete;
    SingletonClass operator=(singletonClass&&) = delete;

protected: // or private
    SingletonClass()
};
```

## 双重校验实现

```cpp
class SingletonClass
{
    static SingletonClass& get_Instance();
private:
    static std::atomic(SingletonClass*) m_instance;
    static std::mutex mtx;
};

SingletonClass& SingletonClass::get_Instance()
{
    SingletonClass *ins = m_instance.load(std::memory_order_consume);
    if(!ins)
    {
        std::lock_guard<std::mutex> lock(mtx);
        ins = m_instance.load(std::memory_order_consume);
        if(!ins)
        {
            ins = new SingletonClass();
            m_instance.store(ins,std::memory_order_release);
        }
    }
}
```

## C++20使用 call_once 实现

```cpp
class SingletonClass
{
public:
    static std::once_flag m_flag;
    static SingletonClass& get_Instance();
    // remember to delete 
private:
    static void initInstance();
    static std::unique_ptr<SingletonClass> m_instance = nullptr;
};

void SingletonClass::initInstance()
{
    m_instance = std::make_unique<SingletonClass>();
}

SingletonClass& SingletonClass::get_Instance()
{
    std::call_once(m_flag, &SingletonClass::initInstance);
    return *m_instance;
}
```

## 单例模式存在的问题

假设有如下的单例模式设计：

```cpp
class Database
{
public:
    virtual int get_population(const std::string& name) = 0;
};

class SingletonDatabase : public Database
{
private:
    SingletonDatabase(){/* read data from database */}
    std::map<std::string,int> capitals_population;
public:
    SingletonDatabse(const SingletonDatabase&) = delete;
    SingletonDatabase operator=(const SingletonDatabase&) = delete;

    static SingletonDatabase& get()
    {
        static SingletonDatabase db;
        return db;
    }

    int get_population(const std::string& name) override
    {
        return capitals_popution[name];
    }
};
```

在如上实现中，我们通过将数据库中的 首都：人口 数据读入我们的 map 成员中保存，并通过接口 get_population 来获取对应首都人口数量。当我们需要一个组件获取总人口时：

```cpp
struct SingletonRecordFinder
{
    int total_population(std::vector<std::string> capitals)
    {
        int result = 0;
        for(auto& capital:capitals)
        {
            result += SingletonDatabase::get().get_population(capital);
        }
        return result;
    }
};
```

但当我们的测试人员需要对 SingletonRecordFinder 接口进行测试时，又因为其完全依赖于 SingletonDatabase，如果想检查新组件就会编写如下测试：

```cpp
TEST(RecordFinderTests, SingletonTotalPopulationTest)
{
    SingletonRecordFinder rf;
    std::vector<std::string> names{"Seoul", "Mexico City"};
    int tp = rf.total_population(names);
    EXPECT_EQ(17500000+17400000, tp);
}
```

这很糟糕，因为我们只想测试 SingletonRecordFinder 组件，但这下我们必须把整个数据库实例读取进内存参与测试，如果我们的目的是测试两者之间是否能够正常运行的集成测试，拿到也说得过去，但是当我们需要的是单元测试时，这就将十分糟糕。

解决方法之一就是设计这个组件的时候需要保存其source来源，而不是直接读取，如下：

```cpp
struct ConfigurableRecordFinder
{
    Database& db;
    explicit ConfigurableRecordFinder(Database& db)
        db{db}
    {}
    int total_population(std::vector<std::string> names)
    {
        int result = 0;
        for(auto& capital: names)
        {
            result += db.get_population(capital);
        }
        return result;
    }
}
```

如此，单元测试时我们可以自行构建用于测试的 Database 虚拟实例，而不是使用我们的物理实例。
