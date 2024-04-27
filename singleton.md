# 单例模式

单例模式是指整个运行的系统中**最多只能有一个**类的实例。  
因为单例模式固有的特性————仅一个实例，所以可以把这部分特性独立出来，写成基类。然后把具体的接口操作可以放在其派生类中。

* [饿汉式分析与实现](#1)  
* [懒汉式分析与实现](#2)  
　　[堆上分配空间](#2.1)    
　　[栈上分配空间-不加锁的懒汉式](#2.2)   
* [各种实现的对比](#3)


<h2 id="1"></h2>

## 饿汉式的分析与实现

饿汉式在类装载时就实例化一个静态实例，所以天生是多线程的。  
以下实现一个饿汉类的基类：

```c
template<typename T>
class Singleton {
public:
    static T *Instance() { return _ins; }

protected:
    Singleton() { garbo;}
    virtual ~Singleton() { } 

private:
    class Garbo {
    public:
        ~Garbo() {
            if (Singleton<T>::_ins) {
                delete Singleton<T>::_ins;
                Singleton<T>::_ins = NULL;
            }   
        }    
    };  
    static T*  _ins;
    static Garbo garbo;
};
template<typename T> T *Singleton<T>::_ins = new T();
template<typename T> typename Singleton<T>::Garbo Singleton<T>::garbo;

```

其中，
- 实例是分配在堆上的，所以需要用户管理其空间的释放。
- 定义一个静态实例 garbo, 保证实例所占内存空间的释放。
- 构造函数中，调用下 garbo, 保证了类对象 garbo 的存在(否则，不会创建对象 garbo, 因为程序中并没有引用的话就不会创建)。


----------------------------------------------------------------
<h2 id="2"></h2>

## 2. 懒汉式分析与实现

<h3 id="2.1"></h3>

### 堆上分配空间

实现一：
```c
template<typename T>
class Singleton {
public:
    static T *Instance();

protected:
    Singleton() { garbo; }

private:
    class Garbo {
    public:
        ~Garbo() {
            if (Singleton<T>::_ins) {
                delete Singleton<T>::_ins;
                Singleton<T>::_ins = NULL;
            }
        }
    };
    static T *_ins;
    static pthread_mutex_t mutex;
    static Garbo garbo;
};
template<typename T> T* Singleton<T>::_ins = NULL;
template<typename T> pthread_mutex_t Singleton<T>::mutex = PTHREAD_MUTEX_INITIALIZER;
template<typename T> typename Singleton<T>::Garbo Singleton<T>::garbo;

template<typename T>
T* Singleton<T>::Instance()
{
    if (NULL == _ins) {
        pthread_mutex_lock(&mutex);
        if (NULL == _ins) {
            _ins = new T();
        }
        pthread_mutex_unlock(&mutex);
    }
    return _ins;
}

```

实现二：
```c
template<typename T>
class Singleton {
public:
    static T *Instance();

protected:
    Singleton() {}
    ~Singleton() { }

private:
    struct SingletonContainer {
        T *_ins;
        SingletonContainer(): _ins(NULL) {}
        ~SingletonContainer() { delete _ins; }
    };  
    static pthread_mutex_t mutex;
    static SingletonContainer SC; 
};
template<typename T> pthread_mutex_t Singleton<T>::mutex = PTHREAD_MUTEX_INITIALIZER;
template<typename T> typename Singleton<T>::SingletonContainer Singleton<T>::SC;

template<typename T>
T* Singleton<T>::Instance()
{
    if (NULL == SC._ins) {
        pthread_mutex_lock(&mutex);
        if (NULL == SC._ins) {
            SC._ins = new T();
        }
        pthread_mutex_unlock(&mutex);
    }
    return SC._ins;
}

```


<h3 id="2.2"></h3>

### 栈上分配空间

```c
#define DISALLOW_COPY_AND_ASSIGN(TypeName) \
        TypeName (const TypeName&);        \
        TypeName& operator= (const TypeName&)

template<typename T>
class Singleton{
public:
    static T* Instance(){
        static T instance;
        return &instance;
    }   
protected:
    Singleton() {}
    virtual ~Singleton() { }

private:
    DISALLOW_COPY_AND_ASSIGN(Singleton);
};

```

------------------------------------------------------------------
<h2 id="3"></h2>

## 3. 各种实现的对比

| 实现          |  优缺点     |
| :-----:       | :--------- |
| 饿汉式 (用栈) | 优点：绝对线程安全;缺点：假如工厂模式，会缓存很多实例，影响效率;无法根据参数创建实例 |
| 懒汉式 (用堆) | 优点：延迟加载,动态创建；缺点：需要同步，引入额外空间                                |
| 懒汉式(用栈)  | 绝对线程安全，并且是动态创建 <推荐>                                                  |


### **线程安全**和**空间释放情况**测试：

```c
class Demo: public Singleton<Demo> {
    friend class Singleton<Demo>;
public:
    void print() { cout << "pid_t: " << pthread_self() << "\tsingleton: " 
                    << this << "\tuniq_val:" << ++uniq_val << endl; }
protected:
    Demo1(): uniq_val(0) {}
private:
    int uniq_val;
};

#define THREAD_FUN_IMPL(n)                       \
    void *thread ## n (void * arg) {             \
        for (int i = 0; i < 5; ++i) {            \
            pthread_mutex_lock(&g_mutex);        \
            Demo ## n::Instance()->print();      \
            pthread_mutex_unlock(&g_mutex);      \
            usleep(1);                           \
        }                                        \
    }

THREAD_FUN_IMPL(1)
THREAD_FUN_IMPL(2)
THREAD_FUN_IMPL(3)
THREAD_FUN_IMPL(4)

pthread_mutex_t g_mutex;

int main()
{
    pthread_t pid11, pid12, pid2, pid3, pid4;
    pthread_mutex_init(&g_mutex, NULL);
    cout << "TEST-lazy-1: (if multi-threads safe)---------------------- " << endl;
    pthread_create(&pid11, NULL, thread1, NULL);
    pthread_create(&pid12, NULL, thread1, NULL);
    pthread_join(pid11, NULL);
    pthread_join(pid12, NULL);

    cout << "TEST-lazy-stack, hungry, lazy-2 pattern---------------------- " << endl;
    pthread_create(&pid2, NULL, thread2, NULL);
    pthread_create(&pid3, NULL, thread3, NULL);
    pthread_create(&pid4, NULL, thread4, NULL);

    pthread_join(pid2, NULL);
    pthread_join(pid3, NULL);
    pthread_join(pid4, NULL);
    pthread_mutex_destroy(&g_mutex);
    return 0;
}

```

![运行结果](https://github.com/JMWY/MyBlog/blob/master/DesignPattern/imgates/singleton.png)
