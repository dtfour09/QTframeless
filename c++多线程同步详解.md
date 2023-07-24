[![img](https://zhuanlan.zhihu.com/p/607028487)
](javascript:void(0))



首发于[硬核编程100式](https://www.zhihu.com/column/hardcore100)



写文章

![点击打开dtfour07的主页](https://pic1.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c)

# Modern C++多线程同步: 由简至繁的三个案例详解

[![minmin](https://pic1.zhimg.com/v2-9853faf82b95f5801d0ab3006e06babc_l.jpg?source=172ae18b)](https://www.zhihu.com/people/minc)

[minmin](https://www.zhihu.com/people/minc)

关注他

88 人赞同了该文章



目录

收起

多线程的两个主要用途

保护共享数据(Protecting shared data from concurrent access)

同步并行操作(Synchronizing concurrent operations)

同步并行操作的三种方法

方法1: 周期性检查条件

方法2: 使用条件变量(conditional variable)

predicate

为什么要在有condition_variable的时候用std::unique_lock?

把mutex传递给conditional variable这个过程中发生了什么?

conditional variable在wait的时候, 底层发生了什么?

无效唤醒(spurious wakeups)

丢失唤醒(Lost Wakeups)

方法3: 使用Future或Promise

Future

Promise

练习题

多线程debug经验

最近又回顾了一下C++多线程看到一篇文章很不错, 翻译一下并且添加一些我自己的见解. 文章来源: [https://chrizog.com/cpp-thread-synchronization](https://link.zhihu.com/?target=https%3A//chrizog.com/cpp-thread-synchronization)

## 多线程的两个主要用途

1. 保护共享数据(Protecting shared data from concurrent access)
2. 同步并行操作(Synchronizing concurrent operations)

## 保护共享数据(Protecting shared data from concurrent access)

一般我们用互斥锁(mutual exclusion, mutex)来保护共享数据. 比如在一些场景下, 有一些线程往共享数据里写数据, 有一些线程从共享数据里读数据. 两个或多个线程读数据完全没问题, 但是我们要防止两个线程同时写数据, 也要防止一个线程在写的同时另一个线程还在读. 否则这样就会发生竞争条件(race condition, 官方翻译是"竞争条件", 但我觉得可以直译成"竞争情况"或者意译成"数据竞争").

## 同步并行操作(Synchronizing concurrent operations)

这篇文章我们主要关注同步并行操作. 同步并行的意思是有可能我们会同时运行不同的事件(event), 我们需要协调他们的发生顺序, 比如一个线程必须要等另一个线程完成它的工作才能进行这个线程本身的下一部分工作.

大部分这样类型的问题都可以总结成生产者和消费者模式(producer-consumer pattern): 消费者需要等待生产者生产数据, 并且等待某种条件成真. 生产者生产数据并且改变那种条件.

## 同步并行操作的三种方法

在同步并行操作, 也就是在生产者和消费者模式里, 让消费者等待生产者设定的某个条件, 可以用三种方法完成:

1. 周期性检查条件(最简单但是最差的实现方法)
2. 使用条件变量(condition variables)
3. 使用futures和promises(理解起来比较复杂, 但有些场景可以很方便)

## 方法1: 周期性检查条件

这是一个最简单但是最不好的方法, 看看它的优缺点:

优点: 容易理解

缺点:

1. 消耗大量资源周期性检查(CPU使用率可能会飙升到100%)
2. 不必要地周期性唤醒线程
3. 垃圾设计

我们从一个简单的生产者消费者案例说起. 假如一个场景我们有一个送快递的线程(生产者), 和一个签收的线程(消费者). 消费者等待生产者的那个"条件", 就是快递是否准备好被接收了.

代码框架可以这样:

```text
struct Packet {
  int id;
};

std::mutex m;
std::queue<Packet> packet_queue;

//生产者 - 生成快递包裹
void parcel_delivery() {}

//消费者 - 签收包裹
void recipient() {}

int main() {
  // 启动生产者和消费者线程
}
```

我们来看看最原始的用互斥锁(mutex)的代码实现怎么写:

```text
#include <chrono>
#include <iostream>
#include <mutex>
#include <queue>
#include <ratio>
#include <thread>

struct Packet {
  int id;
};

std::mutex m;
std::queue<Packet> packet_queue;

//生产者 - 生成快递包裹
void parcel_delivery() {
  // 周期性地生产"包裹"
  int packet_id = 0;
  while (true) {
    // 等待一段时间再投送
    std::this_thread::sleep_for(std::chrono::seconds(1));

    Packet new_packet{packet_id};
    {
      //使用lock_guard锁住这段括号{}以内的共享数据, 走出{}段落后会自动解锁
      //为什么用lock_guard? 这段代码即使出错, throw了, 这个段落结束后也会解锁,
      //从而避免死锁(Dead lock)
      std::lock_guard<std::mutex> l{m};
      std::cout << "Work is done and packet id [" << packet_id << "] is ready"
                << std::endl;
      packet_queue.push(new_packet);
    }
    packet_id++;
  }
}

//消费者 - 签收包裹
void recipient() {
  while (true) {
    {
      //使用lock_guard锁住这段括号{}以内的共享数据, 走出{}段落后会自动解锁
      std::lock_guard<std::mutex> l{m};

      //当共享数据queue里的包裹准备了, 就可以取出包裹了
      if (!packet_queue.empty()) {
        auto new_packet = packet_queue.front();
        packet_queue.pop();
        std::cout << "Received new packet with id: [" << new_packet.id << "]. ["
                  << packet_queue.size() << "] packets remaining" << std::endl;
      }
    }
    //这里消费者就疯狂周期性地加锁并且访问数据, 会导致CPU使用率飙升
    //为了避免这种情况, 每次结束后稍微休息一会, 但这方式治标不治本
    //还是没法避免周期性地唤醒这个线程
    //最倒霉情况还会完整睡上100ms才能拿到数据
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
  }
}

int main(int argc, const char** argv) {
  // 启动生产者和消费者线程
  std::thread producing_thread(parcel_delivery);
  std::thread consuming_thread(recipient);
  producing_thread.join();
  consuming_thread.join();
  return 0;
}
```

输出:

```text
Work is done and packet id [0] is ready
Received new packet with id: [0]. [0] packets remaining
Work is done and packet id [1] is ready
Received new packet with id: [1]. [0] packets remaining
Work is done and packet id [2] is ready
Received new packet with id: [2]. [0] packets remaining
Work is done and packet id [3] is ready
Received new packet with id: [3]. [0] packets remaining
```

## 方法2: 使用条件变量(conditional variable)

看看条件变量的优缺点:

优点:

1. 不浪费CPU资源
2. 条件变量可以多次使用 (和后面要说的futures的一次使用对比)

缺点:

1. 需要考虑无效唤醒(spurious wakeups)
2. 需要考虑丢失唤醒(lost wakeups)

典型使用场景:

网络检测线程: 周期性检查数据, 通知工作线程

文件检查线程: 周期性检查文件, 通知工作线程

总结一下就是需要做通知的时候用. 所以我们的快递包裹和签收也符合这种使用场景, 我们需要快递线程通知签收线程.

同样跟着代码框架:

```text
struct Packet {
  int id;
};

std::mutex m;
std::queue<Packet> packet_queue;

//生产者 - 生成快递包裹
void parcel_delivery() {}

//消费者 - 签收包裹
void recipient() {}

int main() {
  // 启动生产者和消费者线程
}
```

来看看用条件变量(conditional variable)的代码实现:

```text
#include <chrono>
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>

struct Packet {
  int id;
};

std::condition_variable cond_var;
std::mutex m;

std::queue<Packet> packet_queue;

//生产者 - 生成快递包裹
void parcel_delivery() {
  // 周期性地生产"包裹"
  int packet_id = 0;
  while (true) {
    // 等待一段时间再投送
    std::this_thread::sleep_for(std::chrono::seconds(1));
    Packet new_packet{packet_id};
    {
      //使用lock_guard锁住这段括号{}以内的共享数据, 走出{}段落后会自动解锁
      //为什么用lock_guard? 这段代码即使出错, throw了, 这个段落结束后也会解锁,
      //从而避免死锁(Dead lock)
      std::lock_guard<std::mutex> l{m};
      std::cout << "Work is done and packet id [" << packet_id << "] is ready"
                << std::endl;
      packet_queue.push(new_packet);
    }
    packet_id++;
    cond_var.notify_one();
  }
}

//消费者 - 签收包裹
void recipient() {
  while (true) {
    //这里注意我们用了unique_lock, 下面会详细讨论一下
    std::unique_lock<std::mutex> lock{m};

    //这里的lambda函数必须是一个predicate, 也就是必须要返回true/false的函数
    cond_var.wait(lock, []() { return !packet_queue.empty(); });

    auto new_packet = packet_queue.front();
    packet_queue.pop();
    std::cout << "Received new packet with id: [" << new_packet.id << "]. ["
              << packet_queue.size() << "] packets remaining" << std::endl;
  }
}

int main(int argc, const char** argv) {
  // 1: Start two threads: Producer and consumer
  std::thread producing_thread(parcel_delivery);
  std::thread consuming_thread(recipient);
  producing_thread.join();
  consuming_thread.join();
  return 0;
}
```

输出:

```text
Work is done and packet id [0] is ready
Received new packet with id: [0]. [0] packets remaining
Work is done and packet id [1] is ready
Received new packet with id: [1]. [0] packets remaining
Work is done and packet id [2] is ready
Received new packet with id: [2]. [0] packets remaining
Work is done and packet id [3] is ready
Received new packet with id: [3]. [0] packets remaining
Work is done and packet id [4] is ready
Received new packet with id: [4]. [0] packets remaining
```

### predicate

条件变量(conditional variable)的第二个参数应该是一个`predicate`函数, 也就是要返回`True/False`的函数. 一般是用来检测是否符合某种条件(比如数据是否可用)再进行后续操作的. 在我们的例子里:

```text
cond_var.wait(lock, []() { return !packet_queue.empty(); });
```

predicate就是这个lambda函数:

```text
[]() { return !packet_queue.empty(); }
```

### 为什么要在有condition_variable的时候用std::unique_lock?

有以下几个原因：

1. 首先我们不能用纯生的`mutex`, 原因和我们用`lock_guard`一样:

```text
mutex.lock();
cond_var.wait(lock, some_predicate_function);
//如果这中间出错throw了, 那这个mutex就死锁(dead lock)了
mutex.unlock();
```

1. 其次我们用不了`lock_guard`. `lock_guard`只能保护当前范围的`mutex`, 没法把`mutex`传递至`conditional variable`
2. 所以我们要用`unique_lock`封装`mutex`, 它的析构函数(destructor)会自动解锁. 如果中间出错throw了还是可以解锁的. 并且它可以把封装的`mutex`传递给`conditional variable`

### 把mutex传递给conditional variable这个过程中发生了什么?

```text
{
    //锁上mutex
    std::unique_lock<std::mutex> lock{m};

    //这里在等待(wait)的时候, 其实是会解锁mutex, 并且等待条件满足后再重新锁上mutex
    //这样就保证了这个线程不会空转或者等待, 从而不消耗cpu资源
    cond_var.wait(lock, some_predicate_function);

    //操作一些被mutex保护的共享数据

    //手动解锁mutex, 如果不管, 在{}结束的时候也会解锁
    lock.unlock();
}
```

### conditional variable在wait的时候, 底层发生了什么?

我觉得形如上面的操作`lock`, `wait`, `unlock`从表层看还是挺难理解的, 表面上看前面加锁后面解锁, 中间看起来一直锁着, 那个`wait`怎么就不消耗资源了呢?

实际上调用`wait`之后, 里面操作系统层面线程库(operating system threading lib)里的操作是这样的:

1. 解锁mutex
2. 把当前线程放到一个等待队列(queue of wait threads)
3. 当收到通知`notify_one`的时候, 等待队列就会把一个线程释放出来
4. 当收到通知`notify_all`的时候, 等待队列就会把所有线程释放出来
5. 重新锁上mutex

所以这样这个线程才没有占用CPU空转消耗资源.

如果还有兴趣, 再看看C语言这部分可以怎么写:

```text
pthread_mutex_lock(&lock);
    //如果条件不满足, 开始等待
    while (!some_predicate_function)
    {
        pthread_cond_wait(&cond_var, &lock);
        //如果被唤醒, 重新上锁后, 就会运行到这里
        //再次检查while的条件:
        //如果条件不满足, 继续新一轮等待, 锁又会被释放
        //如果条件满足, 才跳过while循环直接进行后续数据操作
    }
    //操作一些被mutex保护的共享数据...
    pthread_mutex_unlock(&lock);
```

### 无效唤醒(spurious wakeups)

这个词`spurious wakeups`没有中文翻译但我觉得可以叫无效唤醒, 就是线程被唤醒之后又发现条件不满足, 再次回到等待状态. 维基的英文解释:

```text
A spurious wakeup happens when a thread wakes up from waiting on a condition variable that's been signaled, only to discover that the condition it was waiting for isn't satisfied.
```

典型情况就会发生在上面提到的`notify_all`中:

我们也都能理解如果只需要通知一个线程的时候, 要调用`notify_one`而不是`notify_all`. 但看看用`notify_all`之后会发生什么:

1. 把所有等待队列里的线程都释放出来
2. 其中一个运气好的线程会拿到锁并且锁上, 检查条件后处理数据
3. (无效步骤)其他剩下的线程虽然被唤醒了, 但是都在等待解锁并进行下面的步骤
4. (无效步骤)等第2步中的线程完成工作解锁后, 第3步的其中一个线程拿到锁
5. (无效步骤)但数据可能已经被第2步的线程处理完了, 这个线程重新检查的条件不满足, 又被加进等待队列(wait queue)
6. (无效步骤)其他所有线程重复第3到第5步的操作

看看那些无效步骤, 结论就是不要闲着没事`notify_all`.

上述都参考于马里兰大学的"Operating System"网课, 如有异议别杠我.

### 丢失唤醒(Lost Wakeups)

还有一点需要注意的是我们可能会丢失一些唤醒信号. 比如我们的生产者消费者模型里:

```text
生产者线程: cond_var.notify_one();

消费者线程: cond_var.wait(lock, []() { return !packet_queue.empty(); });
```

生产者和消费者是在两个线程运行的, 所以我们没法确定他们俩的代码的运行顺序. 有可能生产者运行得比较快先调用了`notify_one`, 但是消费者还没运行到等待这一步. 这一个`notify`的通知就丢失了. 等到下一个`notify`的通知它才能继续运行, 或者也有可能永远等不到下一个通知永远结束不了了.

如果像我们之前C语言里的代码, 没有考虑好的话把`while`写成了`if`, 丢失唤醒(Lost Wakeups)就有可能发生:

```text
pthread_mutex_lock(&lock);
    //这里while改成了if, 可以复现Lost Wakeups
    if (!some_predicate_function)
    {
        //如果运行到这之前notify提前执行了, 这里是不是就永远得不到通知了?
        pthread_cond_wait(&cond_var, &lock);
    }
    pthread_mutex_unlock(&lock);
```

好在现在C++封装了这部分逻辑不容易写错了:

```text
{
    std::unique_lock<std::mutex> lock{m};
    //不需要写while再检查条件了
    cond_var.wait(lock, some_predicate_function);
}
```

所以关键点还在于这个`predicate`检查判断条件, 必须要写好. 我们也可以不把这个判断条件传给`wait`, 比如就这么写`cond_var.wait(lock);`也能编译运行, 但显然很容易出现上述问题.

```text
这部分的知识点终于讲完了, 感觉这几个知识点互相呼应成网状结构, 一次顺下来不一定能搞明白. 值得多看几次.
```

## 方法3: 使用Future或Promise

除了直接使用线程, 我们还可以用高级一点的`Future`或者`Promise`. 没有相关背景知识的话可能还是需要费点力来理解这块. 我们先略扫一下他们的优缺点, 然后一会直接从案例里理解他们.

优点:

1. 不需要直接使用线程(`std::thread`)
2. 方便拿到线程运算结果
3. 没有无效唤醒(spurious wakeups)和丢失唤醒(lost wakeups)的问题

缺点:

每次只能生成一个事件(one-shot events), 不能像以前线程里那样一个线程不断生成数据或者消费数据

常见使用场景原文没说, 我觉得是在对未知的事件做异步计算的场景可能更有用, 比如处理前端用户不同的请求, 或者分布式计算接收不同的计算请求. 有大佬知道的话可以留言讲讲.

### Future

首先我们来看看`std::future`. `std::future`本身是一个容器, 它包含了一种可交给未来计算的内容, 这个内容的计算因为会费时间我们延后运行. 我们可以让`std::future`返回一个值, 比如返回`int`, 可以这么写`std::future<int>`. 也可以让`std::future`单纯做计算不返回任何东西, 就可以这样`std::future<void>`.

刚才提过, `future`和之前两种方法(用`mutex`或`conditional variable`)的不同点就是它是一次性的, 每一个`future`都表示它能做一次计算. 但是好处是我们不需要手动管理线程了, 甚至在我们的代码里都见不到`thread`这个东西, `future`自动把计算请求放到额外的一个线程中计算并且等待结果. 这样代码就更容易理解也更好维护了.

因为`future`的局限性, 光用`future`本身没法完全复现我们之前准备和接收包裹那样的生产者消费者模型案例. 我们来看看下面的代码案例.

先来一个最简单的例子, 看看怎么用`std::async`得到一个`std::future`并且得到计算结果的:

```text
#include <chrono>
#include <future>
#include <iostream>
#include <vector>

struct Packet {
    int id;
};

Packet parcel_delivery() {
    static int packet_id = 0;
    Packet packet {
        packet_id
    };
    packet_id++;
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return packet;
}

int main(int argc,
    const char ** argv) {

    std::future < Packet >future = std::async(std::launch::async, parcel_delivery);

    std::cout << "Wait for delivery" << std::endl;

    auto new_packet = future.get();
    std::cout << "Received new packet with id: " << new_packet.id << std::endl;

    return 0;
}
```

输出:

```text
Wait for delivery
Received new packet with id: 0
```

再改成一个创建多个`std::future`的例子, 也就是隐式创建了多个线程, 看看他们是怎么并行的：

```text
#include <chrono>
#include <future>
#include <iostream>
#include <vector>


struct Packet {
    int id;
};

Packet parcel_delivery() {
    static int packet_id = 0;
    Packet packet {
        packet_id
    };
    packet_id++;
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return packet;
}

int main(int argc,
    const char ** argv) {

    std::vector < std::future < Packet >> list_of_futures;
    for (int i = 0; i < 10; ++i) {
        list_of_futures.emplace_back(std::async(std::launch::async, parcel_delivery));
    }

    std::cout << "Wait for delivery" << std::endl;

    for (auto & future: list_of_futures) {
        auto new_packet = future.get();
        std::cout << "Received new packet with id: " << new_packet.id << std::endl;
    }
    return 0;
}
```

输出:

```text
Wait for delivery
Received new packet with id: 0
Received new packet with id: 9
Received new packet with id: 8
Received new packet with id: 1
Received new packet with id: 2
Received new packet with id: 3
Received new packet with id: 4
Received new packet with id: 5
Received new packet with id: 6
Received new packet with id: 7
```

我们可以看到, 用`std::async`生成了`std::future`, 之后拿到结果, 完全避免了手动使用`mutex`和`thread`.

最后一个问题, 调用`std::async`和普通的多线程的区别在哪? 每次我们调用`std::async`, 就会额外生成一个线程吗?

答案是:

如果我们用了`std::launch::async`flag, 比如`std::async(std::launch::async, parcel_delivery)`, 就会额外生成一个线程.

如果我们用了`std::launch::deferred`flag (推迟的意思), 比如`std::async(std::launch::deferred, parcel_delivery)`, 就不会额外生成一个线程, 只是单纯推迟运算而已.

如果我们什么都不写, 比如`std::async(parcel_delivery)`, 按C++标准, 默认可以用任意一种方式, 但是实际上主流编译器的默认方法都是用`std::launch::async`, 也就是会额外生成一个线程.

更多细节可参考: [https://ddanilov.me/std-async-implementations/#:~:text=The%20C%2B%2B%20standard%20says%3A,it%20shares%20a%20shared%20state](https://link.zhihu.com/?target=https%3A//ddanilov.me/std-async-implementations/%23%3A~%3Atext%3DThe%20C%2B%2B%20standard%20says%3A%2Cit%20shares%20a%20shared%20state)

### Promise

如前面所说, 使用`std::async`(和`std::launch::async`flag), 每生成一个`future`就会额外生成一个线程. 所以简化的`future`的使用场景还是挺有限的. 如果我们就想要一个线程来运行多个`future`, 我们可能就要考虑结合`promise`和`future`了.

`std::promise`和`std::future`刚好相反.

使用`std::future`的时候, 我们把计算函数传给`std::async`创建出一个`std::future`(和一个隐藏的线程).

使用`std::promise`的时候, 我们先创建一个`std::promise`, 通过`std::promise`获取一个`std::future`, 同时声明`std::promise`用来装载计算函数的结果, 最后把它和计算函数一起放进一个显式的线程做计算.

另一种理解方式是, 我们可以把`std::async`理解成一个函数, 它能创建一个`promise`, 把`promise`放进一个线程, 最后返回这个`promise`对应的`future`.

说起来特别绕, 还是直接看下面的最简单的代码案例吧:

```text
#include <chrono>
#include <future>
#include <iostream>
#include <thread>
#include <vector>
struct Packet {
  int id;
};

void parcel_delivery(std::promise<Packet> &&promise) {
  static int id = 0;
  Packet packet{id};
  id++;

  std::this_thread::sleep_for(std::chrono::seconds(1));
  std::cout << "preparing packet with id: " << packet.id << std::endl;

  std::this_thread::sleep_for(std::chrono::seconds(2));
  std::cout << "ready to deliver packet with id: " << packet.id << std::endl;

  promise.set_value(packet);
}

int main(int argc, const char **argv) {

  std::promise<Packet> packet_promise;
  std::future<Packet> future = packet_promise.get_future();
  std::thread delivery_thread(parcel_delivery, std::move(packet_promise));

  std::cout << "Wait for delivery" << std::endl;

  auto new_packet = future.get();
  std::cout << "Received new packet with id: " << new_packet.id << std::endl;

  delivery_thread.join();

  return 0;
}
```

输出:

```text
Wait for delivery
preparing packet with id: 0
ready to deliver packet with id: 0
Received new packet with id: 0
```

通过上面例子大概了解了`std::promise`是怎么一回事之后, 我们就可以尝试一下用`std::promise`和`std::future`复现一下我们之前用`conditional variable`做出的生产者消费者模型了:

```text
#include <chrono>
#include <future>
#include <iostream>
#include <thread>
#include <vector>
struct Packet {
  int id;
};

void parcel_delivery(std::vector<std::promise<Packet>> &&promises) {
  static int id = 0;
  for (auto & p: promises){
    Packet packet{id};
    id++;

    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "preparing packet with id: " << packet.id << std::endl;

    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "ready to deliver packet with id: " << packet.id << std::endl;

    p.set_value(packet);
  }
}

int main(int argc, const char **argv) {
  std::vector<std::future<Packet>> futures_list;
  std::vector<std::promise<Packet>> packet_promises;
  for (int i=0; i<5; ++i)
  {
      packet_promises.emplace_back(std::promise<Packet>());
  }

  for(auto & p : packet_promises){
    futures_list.emplace_back(p.get_future());
  }

  std::thread delivery_thread(parcel_delivery, std::move(packet_promises));

  std::cout << "Wait for delivery" << std::endl;

  for (auto & packet_future: futures_list)
  {
    auto new_packet = packet_future.get();
    std::cout << "Received new packet with id: " << new_packet.id << std::endl;
  }

  delivery_thread.join();

  return 0;
}
```

输出:

```text
Wait for delivery
preparing packet with id: 0
ready to deliver packet with id: 0
Received new packet with id: 0
preparing packet with id: 1
ready to deliver packet with id: 1
Received new packet with id: 1
preparing packet with id: 2
ready to deliver packet with id: 2
Received new packet with id: 2
preparing packet with id: 3
ready to deliver packet with id: 3
Received new packet with id: 3
preparing packet with id: 4
ready to deliver packet with id: 4
Received new packet with id: 4
```

## 练习题

最后, 我们用上面学到的知识来做一个练习题吧.

题目: 创建三个线程, 其中一个线程能持续输出"0", 一个线程能持续输出奇数, 一个线程能持续输出偶数, 要求保证输出的结果是按这样的顺序: "0 1 0 2 0 3 0 4 0 5..."

答案:

```text
#include <chrono>
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>


//print_zero
//print_odd
//print_even
//print: 0 1 0 2 0 3 0 4 0 5...
std::condition_variable notify_zero;
std::condition_variable notify_odd;
std::condition_variable notify_even;

bool print_zero_can_start = true;
bool print_odd_can_start = false;
bool print_even_can_start = false;

std::mutex m;


void print_zero() {
    int counter = 0;

    while (true) {
        std::this_thread::sleep_for(std::chrono::seconds(1));//comment this to see whether it actually works

        std::unique_lock<std::mutex> lock{m};
        notify_zero.wait(lock, []() { return print_zero_can_start; });
        counter++;
        std::cout << 0 << std::endl;
        if (counter%2==1){
                print_odd_can_start = true;
        }
        if (counter%2==0){
                print_even_can_start = true;
        }
        print_zero_can_start = false;
        lock.unlock();

        if (counter%2==1){
                notify_odd.notify_one();
        }
        if (counter%2==0){
                notify_even.notify_one();
        }
    }
}

void print_odd() {
    int odd_counter = 1;
    while (true) {
        std::unique_lock<std::mutex> lock{m};
        notify_odd.wait(lock, []() { return print_odd_can_start; });

        std::cout << odd_counter << std::endl;
        odd_counter += 2;
        print_odd_can_start = false;
        print_zero_can_start = true;
        lock.unlock();

        notify_zero.notify_one();
    }
}

void print_even() {
    int even_counter = 2;
    while (true) {
        std::unique_lock<std::mutex> lock{m};
        notify_even.wait(lock, []() { return print_even_can_start; });

        std::cout << even_counter << std::endl;
        even_counter += 2;
        print_even_can_start = false;
        print_zero_can_start = true;
        lock.unlock();

        notify_zero.notify_one();
    }
}

int main(int argc, const char **argv) {
    std::thread thread0(print_zero);
    std::thread thread1(print_odd);
    std::thread thread2(print_even);
    thread0.join();
    thread1.join();
    thread2.join();
    return 0;
}
```

## 多线程debug经验

最后记录一些多线程debug的经验吧(虽然我写多线程的经验也不多). 一般多线程/多进程的代码运行起来都比较费时, 如果出现因为并行导致的bug那可能是非常随机的, 有时候跑10多分钟可能才随机出现一个奇怪的bug, 之后可能再重新跑半小时都没有bug.

而debug的关键就是持续复现问题. 所以我们可以尽量移除耗时不相关的代码, 并且砍掉大部分没问题的数据, 获得最简化的多线程代码和最简化的数据来复现问题. 最后再创造出最极端的环境, 比如原本3个线程跑10次数据, 改成10个线程跑100次数据.

在这种极限环境, 如果能更容易复现问题, 解决起来就更方便了.

编辑于 2023-02-17 22:12・IP 属地英国

「真诚赞赏，手留余香」

赞赏

还没有人赞赏，快来当第一个赞赏的人吧！

[C++](https://www.zhihu.com/topic/19584970)

[Modern C++](https://www.zhihu.com/topic/20744506)

[多线程](https://www.zhihu.com/topic/19619463)

赞同 882 条评论

分享

喜欢收藏申请转载



赞同 88

分享

![img](https://pic1.zhimg.com/v2-abed1a8c04700ba7d72b45195223e0ff_l.jpg?source=32738c0c)

评论千万条，友善第一条

2 条评论

默认

最新

[![万万没想到](https://pic1.zhimg.com/v2-3af80baa440468b682bffe718d9c8cff_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/4fa7199e4839b1356b15569ee500fc67)

[万万没想到](https://www.zhihu.com/people/4fa7199e4839b1356b15569ee500fc67)



promise不能中途cancel，不能加入超时参数，代码不可预期。

02-18 · IP 属地江苏

回复1

[![王浩](https://picx.zhimg.com/5ba75c362b7ee45db7f21c6746dfb423_l.jpg?source=06d4cd63)](https://www.zhihu.com/people/9f404d780dde9d54998caf432d4e6e34)

[王浩](https://www.zhihu.com/people/9f404d780dde9d54998caf432d4e6e34)



很棒，不错

03-18 · IP 属地广东

回复赞

### 文章被以下专栏收录

- [![硬核编程100式](https://picx.zhimg.com/v2-21f59fb09180f0b5cf352a621406d730_l.jpg?source=172ae18b)](https://www.zhihu.com/column/hardcore100)

- ## [硬核编程100式](https://www.zhihu.com/column/hardcore100)

- 想到什么写什么

### 推荐阅读

- # C++ 多线程（四）：实现一个功能完整的线程池

- 今天我们来聊一聊异步编程的知识。在分布式系统中，一个功能完整的线程池类是一切代码的前提。 一个『合格』的线程池该具备哪些功能？首先，很自然地想到『线程池类里该有个线程对象的集合…

- 自由技艺发表于C++ 系...

- ![C++ 多线程（一）：生产者 - 消费者模型](https://pic1.zhimg.com/v2-5d8e0561ed34d61c18ab2766e220a25b_250x0.jpg?source=172ae18b)

- # C++ 多线程（一）：生产者 - 消费者模型

- 自由技艺发表于C++ 系...

- ![C++标准库多线程简介Part1](https://picx.zhimg.com/v2-0f6f8550bc75e2892c81cd709eddf56b_250x0.jpg?source=172ae18b)

- # C++标准库多线程简介Part1

- 吉良吉影

- # C++多线程编程：基于现代C++的多线程库

- 其余内容见： 深度学习可好玩了：C++高性能编程：概览本文是《c++并发编程实战》第二版 网址的读书笔记。c++多线程历史首先，c++标准目前还是只支持了多线程，多进程需要用操作系统API。 C+…

- 深度学习可好玩了