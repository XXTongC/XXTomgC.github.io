# 观察者模式(Observer)

观察者模式的核心在于当被观察者的状态发生变化时，观察者需要获取状态变化的通知，所以被观察者需要维护一个观察者的集合并负责通知。

对于观察者，需要设计其通知方法，实现相关类的观察者需要继承这个基类，并实现其关于通知的纯虚函数：

```cpp
template<typename T>
struct Observer
{
    virtual void field_changed(T& source, const std::string& field_name) = 0;
};
```

对于被观察者，其需要继承可被观察类的属性：

```cpp
template<typename T>
struct Observable
{
    void notify(T& source, const std::string& name){
        for(auto& obs : observers)
        {
            obs->field_changed(source, name);
        }
    }
    void subscribe(Observer<T>* f) {observers.emplace_back(f);}
    void unsubscribe(Observer<T>* f){...}

private:
    // 因为涉及到注销观察者，
    // 可以根据情况用不同的容器去维护观察者
    std::vector<Observer<T>*> observers;
};
```

## 示例

```cpp
struct Person : Observable<Person>
{
    void set_age(int value){
        if(age==value) return;
        age = value;
        notify(*this,"age");
        
    }
private:
    int age;

};

struct PersonObserver : Observer<Person>
{
    void field_changed(T& source, const std::string& name) override
    {
        if(name=="age")
        {
            std::cout << name << " changed into "<< source.get_age() << ".\n";
        }
    }
}
```

此时当我们调用了 set_age() 后，函数调用如下：

set_age() -> notify() -> field_changed()

其一种视图的设计方法是使用装饰器模式，要使用视图，可以使用普通属性（甚至是公共访问域），使对象保持简单且没有任何额外的行为：

```cpp
template<typename T>
struct Observable
{
    // 使用了Boost.Signals2库
    signal<void(T&,const std::string&)> property_changed;
}

struct Person{
    std::string name;
};

struct PersonView : Obervable<Person>
{
    explicit PersonView(const Person& person)
        :person{person}
    {

    }
    std::string& get_name()
    {
        return person.name;
    }

    void set_name(const std::string& value)
    {
        if(value==person.name) return;
        person.name = value;
        property_changed(person,"name");
    }
protected:
    Person& person;
}
```