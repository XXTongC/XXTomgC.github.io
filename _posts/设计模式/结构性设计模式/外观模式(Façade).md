# 外观模式(Façade)

在现代软件系统中，我们经常需要与复杂的子系统交互。这些子系统可能包含大量的类、复杂的API和繁琐的初始化流程。直接使用这些子系统会导致客户端代码变得复杂且难以维护。

外观模式(Facade)通过提供一个统一的高层接口，使得子系统更加容易使用。它隐藏了子系统的复杂性，为客户端提供了一个简单的入口点，同时不会限制那些需要直接访问子系统的高级用户。

## 外观模式的典型结构

```txt
┌─────────────────┐
│     Client      │
└────────┬────────┘
         │ 使用
         ↓
┌─────────────────┐
│     Facade      │ (外观类)
├─────────────────┤
│- subsystem1     │
│- subsystem2     │
│- subsystem3     │
├─────────────────┤
│+ operation1()   │ (简化的高层接口)
│+ operation2()   │
└────┬───┬───┬────┘
     │   │   │ 委托调用
     ↓   ↓   ↓
┌────┴───┴───┴──────────────────────────────┐
│          复杂子系统                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │SubsystemA│  │SubsystemB│  │Subsystem │ │
│  │          │  │          │  │    C       │
│  │+ method()│  │+ method()│  │+ method()│ │
│  └──────────┘  └──────────┘  └──────────┘ │
│                                           │
│  ┌──────────┐  ┌──────────┐               │
│  │SubsystemD│  │SubsystemE│   ...         │
│  │          │  │          │               │
│  │+ method()│  │+ method()│               │
│  └──────────┘  └──────────┘               │
└───────────────────────────────────────────┘
```

| 角色 | 职责 | 说明 |
|------|------|------|
| **Facade** | 外观类 | 提供简化的接口，将客户端请求委托给适当的子系统对象 |
| **Subsystems** | 子系统类 | 实现子系统功能，处理Facade对象分配的工作，但不知道Facade的存在 |
| **Client** | 客户端 | 通过Facade接口与子系统通信，无需直接访问子系统 |

其工作流程大致如下：

```txt
客户端调用 → Facade.operation()
                    ↓
            ┌───────┴───────┐
            ↓               ↓
    SubsystemA.method1() SubsystemB.method2()
            ↓               ↓
    SubsystemC.method3() SubsystemD.method4()
            ↓               ↓
            └───────┬───────┘
                    ↓
            返回统一结果给客户端
```

外观模式的实现方式通常有以下几种：

* 简单外观 提供对子系统的直接访问，仅仅是对现有接口的简单包装。
* 复杂外观 封装复杂的初始化流程和多步骤操作，可能包含状态管理和错误处理。
* 分层外观 为子系统的不同层次提供多个外观，形成外观的层次结构。

## 基础外观模式

考虑一个家庭影院系统，它包含多个子系统：DVD播放器、投影仪、音响系统、灯光控制等。直接操作这些子系统需要执行一系列复杂的步骤：

```cpp
// 子系统类
class DVDPlayer
{
public:
    void on() { std::cout << "DVD Player on\n"; }
    void off() { std::cout << "DVD Player off\n"; }
    void play(const std::string& movie) 
    { 
        std::cout << "Playing movie: " << movie << "\n"; 
    }
    void stop() { std::cout << "Stopping DVD\n"; }
};

class Projector
{
public:
    void on() { std::cout << "Projector on\n"; }
    void off() { std::cout << "Projector off\n"; }
    void wideScreenMode() { std::cout << "Projector in widescreen mode\n"; }
};

class SoundSystem
{
public:
    void on() { std::cout << "Sound System on\n"; }
    void off() { std::cout << "Sound System off\n"; }
    void setVolume(int level) 
    { 
        std::cout << "Sound System volume set to " << level << "\n"; 
    }
    void setSurroundSound() { std::cout << "Surround sound on\n"; }
};

class Lights
{
public:
    void dim(int level) 
    { 
        std::cout << "Lights dimmed to " << level << "%\n"; 
    }
    void on() { std::cout << "Lights on\n"; }
};

// 外观类
class HomeTheaterFacade
{
private:
    DVDPlayer& dvd;
    Projector& projector;
    SoundSystem& soundSystem;
    Lights& lights;

public:
    HomeTheaterFacade(DVDPlayer& dvd, Projector& proj, 
                      SoundSystem& sound, Lights& lights)
        : dvd(dvd), projector(proj), soundSystem(sound), lights(lights)
    {}

    void watchMovie(const std::string& movie)
    {
        std::cout << "Get ready to watch a movie...\n";
        lights.dim(10);
        projector.on();
        projector.wideScreenMode();
        soundSystem.on();
        soundSystem.setSurroundSound();
        soundSystem.setVolume(5);
        dvd.on();
        dvd.play(movie);
    }

    void endMovie()
    {
        std::cout << "Shutting down movie theater...\n";
        dvd.stop();
        dvd.off();
        soundSystem.off();
        projector.off();
        lights.on();
    }
};
```

使用外观模式后，客户端代码变得非常简洁：

```cpp
DVDPlayer dvd;
Projector projector;
SoundSystem soundSystem;
Lights lights;

HomeTheaterFacade homeTheater(dvd, projector, soundSystem, lights);

// 简单的接口调用
homeTheater.watchMovie("Inception");
// Get ready to watch a movie...
// Lights dimmed to 10%
// Projector on
// Projector in widescreen mode
// Sound System on
// Surround sound on
// Sound System volume set to 5
// DVD Player on
// Playing movie: Inception

homeTheater.endMovie();
// Shutting down movie theater...
// Stopping DVD
// DVD Player off
// Sound System off
// Projector off
// Lights on
```

## 带状态管理的外观

在实际应用中，外观类可能需要管理状态和处理错误。考虑一个数据库连接外观，它封装了连接池、事务管理和查询执行：

```cpp
#include <memory>
#include <string>
#include <vector>
#include <stdexcept>

// 子系统类
class ConnectionPool
{
private:
    int maxConnections;
    int activeConnections = 0;

public:
    explicit ConnectionPool(int max) : maxConnections(max) {}
    
    bool acquireConnection()
    {
        if (activeConnections < maxConnections)
        {
            ++activeConnections;
            std::cout << "Connection acquired. Active: " << activeConnections << "\n";
            return true;
        }
        return false;
    }

    void releaseConnection()
    {
        if (activeConnections > 0)
        {
            --activeConnections;
            std::cout << "Connection released. Active: " << activeConnections << "\n";
        }
    }

    int getActiveConnections() const { return activeConnections; }
};

class TransactionManager
{
private:
    bool inTransaction = false;

public:
    void begin()
    {
        if (inTransaction)
            throw std::runtime_error("Transaction already in progress");
        inTransaction = true;
        std::cout << "Transaction started\n";
    }

    void commit()
    {
        if (!inTransaction)
            throw std::runtime_error("No active transaction");
        std::cout << "Transaction committed\n";
        inTransaction = false;
    }

    void rollback()
    {
        if (!inTransaction)
            throw std::runtime_error("No active transaction");
        std::cout << "Transaction rolled back\n";
        inTransaction = false;
    }

    bool isActive() const { return inTransaction; }
};

class QueryExecutor
{
public:
    std::vector<std::string> execute(const std::string& query)
    {
        std::cout << "Executing query: " << query << "\n";
        // 模拟查询结果
        return {"result1", "result2", "result3"};
    }
};

// 外观类
class DatabaseFacade
{
private:
    ConnectionPool connectionPool;
    TransactionManager transactionManager;
    QueryExecutor queryExecutor;
    bool connected = false;

public:
    explicit DatabaseFacade(int maxConnections = 10)
        : connectionPool(maxConnections)
    {}

    bool connect()
    {
        if (connected)
        {
            std::cout << "Already connected\n";
            return true;
        }

        if (connectionPool.acquireConnection())
        {
            connected = true;
            std::cout << "Database connected\n";
            return true;
        }

        std::cout << "Failed to connect: no available connections\n";
        return false;
    }

    void disconnect()
    {
        if (!connected)
            return;

        if (transactionManager.isActive())
        {
            std::cout << "Rolling back active transaction before disconnect\n";
            transactionManager.rollback();
        }

        connectionPool.releaseConnection();
        connected = false;
        std::cout << "Database disconnected\n";
    }

    std::vector<std::string> executeQuery(const std::string& query)
    {
        if (!connected)
            throw std::runtime_error("Not connected to database");

        return queryExecutor.execute(query);
    }

    void executeTransaction(const std::vector<std::string>& queries)
    {
        if (!connected)
            throw std::runtime_error("Not connected to database");

        try
        {
            transactionManager.begin();

            for (const auto& query : queries)
            {
                queryExecutor.execute(query);
            }

            transactionManager.commit();
        }
        catch (const std::exception& e)
        {
            std::cout << "Error during transaction: " << e.what() << "\n";
            transactionManager.rollback();
            throw;
        }
    }

    ~DatabaseFacade()
    {
        if (connected)
            disconnect();
    }
};
```

使用示例：

```cpp
DatabaseFacade db(5);

// 简单查询
if (db.connect())
{
    auto results = db.executeQuery("SELECT * FROM users");
    for (const auto& result : results)
    {
        std::cout << "Result: " << result << "\n";
    }
}
// Connection acquired. Active: 1
// Database connected
// Executing query: SELECT * FROM users
// Result: result1
// Result: result2
// Result: result3

// 事务操作
std::vector<std::string> transactionQueries = {
    "INSERT INTO users VALUES (1, 'Alice')",
    "UPDATE accounts SET balance = 1000 WHERE user_id = 1",
    "DELETE FROM temp_data WHERE id < 100"
};

db.executeTransaction(transactionQueries);
// Transaction started
// Executing query: INSERT INTO users VALUES (1, 'Alice')
// Executing query: UPDATE accounts SET balance = 1000 WHERE user_id = 1
// Executing query: DELETE FROM temp_data WHERE id < 100
// Transaction committed

db.disconnect();
// Database disconnected
// Connection released. Active: 0
```

## 模块化外观

对于需要支持不同后端实现的场景,可以使用模板来创建灵活的外观。考虑一个日志系统外观，它可以支持不同的日志后端：

```cpp
#include <iostream>
#include <fstream>
#include <sstream>
#include <chrono>
#include <iomanip>

// 日志级别
enum class LogLevel
{
    DEBUG,
    INFO,
    WARNING,
    ERROR
};

// 不同的日志后端
class ConsoleLogger
{
public:
    void write(const std::string& message)
    {
        std::cout << message << std::endl;
    }
};

class FileLogger
{
private:
    std::ofstream file;

public:
    explicit FileLogger(const std::string& filename)
    {
        file.open(filename, std::ios::app);
        if (!file.is_open())
            throw std::runtime_error("Failed to open log file");
    }

    void write(const std::string& message)
    {
        if (file.is_open())
        {
            file << message << std::endl;
            file.flush();
        }
    }

    ~FileLogger()
    {
        if (file.is_open())
            file.close();
    }
};

// 格式化器
class LogFormatter
{
public:
    static std::string format(LogLevel level, const std::string& message)
    {
        std::ostringstream oss;
        
        // 添加时间戳
        auto now = std::chrono::system_clock::now();
        auto time = std::chrono::system_clock::to_time_t(now);
        oss << std::put_time(std::localtime(&time), "%Y-%m-%d %H:%M:%S");
        
        // 添加日志级别
        oss << " [" << levelToString(level) << "] ";
        
        // 添加消息
        oss << message;
        
        return oss.str();
    }

private:
    static std::string levelToString(LogLevel level)
    {
        switch (level)
        {
            case LogLevel::DEBUG:   return "DEBUG";
            case LogLevel::INFO:    return "INFO";
            case LogLevel::WARNING: return "WARNING";
            case LogLevel::ERROR:   return "ERROR";
            default:                return "UNKNOWN";
        }
    }
};

// 过滤器
class LogFilter
{
private:
    LogLevel minLevel;

public:
    explicit LogFilter(LogLevel level = LogLevel::DEBUG)
        : minLevel(level)
    {}

    bool shouldLog(LogLevel level) const
    {
        return level >= minLevel;
    }

    void setMinLevel(LogLevel level)
    {
        minLevel = level;
    }
};

// 模板化的日志外观
template<typename Backend>
class LoggingFacade
{
private:
    Backend backend;
    LogFormatter formatter;
    LogFilter filter;

public:
    template<typename... Args>
    explicit LoggingFacade(Args&&... args)
        : backend(std::forward<Args>(args)...)
        , filter(LogLevel::INFO)
    {}

    void setLogLevel(LogLevel level)
    {
        filter.setMinLevel(level);
    }

    void debug(const std::string& message)
    {
        log(LogLevel::DEBUG, message);
    }

    void info(const std::string& message)
    {
        log(LogLevel::INFO, message);
    }

    void warning(const std::string& message)
    {
        log(LogLevel::WARNING, message);
    }

    void error(const std::string& message)
    {
        log(LogLevel::ERROR, message);
    }

    template<typename... Args>
    void debug(const std::string& format, Args&&... args)
    {
        log(LogLevel::DEBUG, formatMessage(format, std::forward<Args>(args)...));
    }

    template<typename... Args>
    void info(const std::string& format, Args&&... args)
    {
        log(LogLevel::INFO, formatMessage(format, std::forward<Args>(args)...));
    }

    template<typename... Args>
    void warning(const std::string& format, Args&&... args)
    {
        log(LogLevel::WARNING, formatMessage(format, std::forward<Args>(args)...));
    }

    template<typename... Args>
    void error(const std::string& format, Args&&... args)
    {
        log(LogLevel::ERROR, formatMessage(format, std::forward<Args>(args)...));
    }

private:
    void log(LogLevel level, const std::string& message)
    {
        if (filter.shouldLog(level))
        {
            std::string formattedMessage = formatter.format(level, message);
            backend.write(formattedMessage);
        }
    }

    template<typename... Args>
    std::string formatMessage(const std::string& format, Args&&... args)
    {
        std::ostringstream oss;
        formatHelper(oss, format, std::forward<Args>(args)...);
        return oss.str();
    }

    template<typename T, typename... Args>
    void formatHelper(std::ostringstream& oss, const std::string& format, 
                     T&& value, Args&&... args)
    {
        size_t pos = format.find("{}");
        if (pos != std::string::npos)
        {
            oss << format.substr(0, pos) << value;
            formatHelper(oss, format.substr(pos + 2), std::forward<Args>(args)...);
        }
        else
        {
            oss << format;
        }
    }

    void formatHelper(std::ostringstream& oss, const std::string& format)
    {
        oss << format;
    }
};
```

使用示例：

```cpp
// 使用控制台日志
LoggingFacade<ConsoleLogger> consoleLog;
consoleLog.setLogLevel(LogLevel::DEBUG);

consoleLog.debug("Application started");
// 2024-01-15 10:30:45 [DEBUG] Application started

consoleLog.info("User {} logged in from IP {}", "Alice", "192.168.1.100");
// 2024-01-15 10:30:46 [INFO] User Alice logged in from IP 192.168.1.100

consoleLog.warning("Memory usage at {}%", 85);
// 2024-01-15 10:30:47 [WARNING] Memory usage at 85%

consoleLog.error("Failed to connect to database");
// 2024-01-15 10:30:48 [ERROR] Failed to connect to database

// 使用文件日志
LoggingFacade<FileLogger> fileLog("application.log");
fileLog.setLogLevel(LogLevel::INFO);

fileLog.debug("This won't be logged");  // 被过滤器过滤
fileLog.info("This will be logged to file");
fileLog.error("Critical error occurred");
```