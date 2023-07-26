# Qt多线程实现的方式

## 一，QThread类的run

通过新建继承一个QThread类，重写虚函数run，启动线程。

代码

```
 class WorkerThread : public QThread
  {
      Q_OBJECT
      void run() override {
          QString result;
          /* ... here is the expensive or blocking operation ... */
          emit resultReady(result);
      }
  signals:
      void resultReady(const QString &s);
  };
 
  void MyObject::startWorkInAThread()
  {
      WorkerThread *workerThread = new WorkerThread(this);
      connect(workerThread, &WorkerThread::resultReady, this, &MyObject::handleResults);
      connect(workerThread, &WorkerThread::finished, workerThread, &QObject::deleteLater);
      workerThread->start();
  }
```

特点：

1、优点：可以通过信号槽与外界进行通信。
2、缺点：1每次新建一个线程都需要继承QThread，实现一个新类，使用不太方便。
要自己进行资源管理，线程释放和删除。并且频繁的创建和释放会带来比较大的内存开销。
3、适用场景：QThread适用于那些常驻内存的任务。

## 二，通过QThread类的moveToThread

创建一个继承object的类，然乎new一个Qthread，并把创建的obeject类movetothread到创建好的子线程中，然后start子线程。

代码：

```
class Worker : public QObject
  {
      Q_OBJECT
 
  public slots:
      void doWork(const QString &parameter) {
          QString result;
          /* ... here is the expensive or blocking operation ... */
          emit resultReady(result);
      }
 
  signals:
      void resultReady(const QString &result);
  };
 
  class Controller : public QObject
  {
      Q_OBJECT
      QThread workerThread;
  public:
      Controller() {
          Worker *worker = new Worker;
          worker->moveToThread(&workerThread);
          connect(&workerThread, &QThread::finished, worker, &QObject::deleteLater);
          connect(this, &Controller::operate, worker, &Worker::doWork);
          connect(worker, &Worker::resultReady, this, &Controller::handleResults);
          workerThread.start();
      }
      ~Controller() {
          workerThread.quit();
          workerThread.wait();
      }
  public slots:
      void handleResults(const QString &);
  signals:
      void operate(const QString &);
  };
```

特点

MovetoThreadTest.cpp 中用到了窗体中的几个按钮，在如用代码是首相创建一个窗体，在窗体添加按钮，然后跟据按钮的名字进行连接即可；

主线程如果要在子线程中运行计算必须通过发信号的方式调用，或者通过控件的信号；需要说明在创建连接时如果第五的参数为DirectConnection时，调用的槽函数是在主线程中运行；如果想通过子线程向主线程调用方法，也必须通过发信号的方式触发主线程的函数。

Qt::AutoConnection，t::DirectConnection，t::QueuedConnection，t::BlockingQueuedConnection，t::UniqueConnection

Qt::AutoCompatConnection这里面一共有六种方式。

前两种比较相似，都是同一线程之间连接的方式，不同的是Qt::AutoConnection是系统默认的连接方式。这种方式连接的时候，槽不是马上被执行的，而是进入一个消息队列，待到何时执行就不是我们可以知道的了，当信号和槽不是同个线程，会使用第三种QT::QueueConnection的链接方式。如果信号和槽是同个线程，调用第二种Qt::DirectConnection链接方式。
第二种Qt::DirectConnection是直接连接，也就是只要信号发出直接就到槽去执行，无论槽函数所属对象在哪个线程，槽函数都在发射信号的线程内执行，一旦使用这种连接，槽将会不在线程执行！。
第三种Qt::QueuedConnection和第四种Qt::BlockingQueuedConnection是相似的，都是可以在不同进程之间进行连接的，不同的是，这里第三种是在对象的当前线程中执行,并且是按照队列顺序执行。当当前线程停止，就会等待下一次启动线程时再按队列顺序执行 ，等待QApplication::exec()或者线程的QThread::exec()才执行相应的槽，就是说：当控制权回到接受者所依附线程的事件循环时，槽函数被调用，而且槽函数在接收者所依附线程执行，使用这种连接，槽会在线程执行。
第四种Qt::BlockingQueuedConnection是（必须信号和曹在不同线程中，否则直接产生死锁）这个是完全同步队列只有槽线程执行完才会返回，否则发送线程也会等待，相当于是不同的线程可以同步起来执行。
第五种Qt::UniqueConnection跟默认工作方式相同，只是不能重复连接相同的信号和槽；因为如果重复链接就会导致一个信号发出，对应槽函数就会执行多次。
第六种Qt::AutoCompatConnection是为了连接QT4 到QT3的信号槽机制兼容方式，工作方式跟Qt::AutoConnection一样。显然这里我们应该选择第三种方式，我们不希望子线程没结束主线程还要等，我们只是希望利用这个空闲时间去干别的事情，当子线程执行完了，只要发消息给主线程就行了，到时候主线程会去响应。

moveToThread对比传统子类化Qthread更灵活，仅需要把你想要执行的代码放到槽，movetothread这个object到线程，然后拿一个信号连接到这个槽就可以让这个槽函数在线程里执行。可以说，movetothread给我们编写代码提供了新的思路，当然不是说子类化qthread不好，只是你应该知道还有这种方式去调用线程。

轻量级的函数可以用movethread，多个短小精悍能返回快速的线程函数适用 ，无需创建独立线程类，例如你有20个小函数要在线程内做, 全部扔给一个QThread。而我觉得movetothread和子类化QThread的区别不大，更可能是使用习惯引导。又**或者你一开始没使用线程，但是后边发觉这些代码还是放线程比较好，如果用子类化QThread的方法重新设计代码，将会有可能让你把这一段推到重来，这个时候，moveThread的好处就来了，你可以把这段代码的从属着movetothread，把代码移到槽函数，用信号触发它就行了**。其它的话movetothread它的效果和子类化QThread的效果是一样的，槽就相当于你的run()函数，你往run()里塞什么代码，就可以往槽里塞什么代码，子类化QThread的线程只可以有一个入口就是run()，而movetothread就有很多触发的入口。

## **三、QRunnalble的run**

Qrunnable是所有可执行对象的基类。我们可以继承Qrunnable，并重写虚函数void QRunnable::run () 。我们可以用QThreadPool让我们的一个QRunnable对象在另外的线程中运行，如果autoDelete()返回true(默认)，那么QThreadPool将会在run()运行结束后自动删除Qrunnable对象。可以调用void QRunnable::setAutoDelete ( bool autoDelete )更改auto-deletion标记。需要注意的是，必须在调用QThreadPool::start()之前设置，在调用QThreadPool::start()之后设置的结果是未定义的。

实现方法：

1、继承QRunnable。和QThread使用一样， 首先需要将你的线程类继承于QRunnable。

2、重写run函数。还是和QThread一样，需要重写run函数，run是一个纯虚函数，必须重写。

3、使用QThreadPool启动线程

代码

```

class Runnable:public QRunnable
{
       //Q_OBJECT   注意了，Qrunnable不是QObject的子类。
public:
       Runnable();
       ~Runnable();
       void run();
};
 
 
Runnable::Runnable():QRunnable()
{
 
}
 
Runnable::~Runnable()
{
       cout<<"~Runnable()"<<endl;
}
 
void Runnable::run()
{
       cout<<"Runnable::run()thread :"<<QThread::currentThreadId()<<endl;
       cout<<"dosomething ...."<<endl;
}
int main(int argc, char *argv[])
{
       QCoreApplication a(argc, argv);
       cout<<"mainthread :"<<QThread::currentThreadId()<<endl;
       Runnable runObj;
       QThreadPool::globalInstance()->start(&runObj);
       returna.exec();
}
```

特点：

优点：无需手动释放资源，QThreadPool启动线程执行完成后会自动释放。
缺点：不能使用信号槽与外界通信。
适用场景：QRunnable适用于线程任务量比较大，需要频繁创建线程。QRunnable能有效减少内存开销。



## **四、QtConcurrent的run**

Concurrent是并发的意思，QtConcurrent是一个命名空间，提供了一些高级的 API，使得在编写多线程的时候，无需使用低级线程原语，如读写锁，等待条件或信号。使用QtConcurrent编写的程序会根据可用的处理器内核数自动调整使用的线程数。这意味着今后编写的应用程序将在未来部署在多核系统上时继续扩展。

QtConcurrent::run能够方便快捷的将任务丢到子线程中去执行，无需继承任何类，也不需要重写函数，使用非常简单。详见前面的文章介绍，这里不再赘述。

需要注意的是，由于该线程取自全局线程池QThreadPool，函数不能立马执行，需要等待线程可用时才会运行。

实现方法：

1、首先在.pro文件中加上以下内容：QT += concurrent

2、包含头文件#include ，然后就可以使用QtConcurrent了

QFuture<void> fut1 = QtConcurrent::run(func, QString("Thread 1")); fut1.waitForFinished();

代码

```

#include <QtCore/QCoreApplication>
#include <QDebug>
#include <QThread>
#include <QString>
#include <QtConcurrent/QtConcurrentRun>
#include <QTime>
#include<opencv2\opencv.hpp>
#include"XxwImgOp.h"
#ifdef _DEBUG
#pragma comment(lib,".\\XxwImgOpd.lib")
#else
#pragma comment(lib,".\\XxwImgOp.lib")
#endif // _DEBUG
 
using namespace QtConcurrent;
 
XxwImgOp xxwImgOp;
cv::Mat src = cv::imread("1.bmp", 0);
cv::Mat  dst, dst1, dst2;
 
void hello(cv::Mat src)
{
	qDebug() << "-----------" << QTime::currentTime()<<"------------------------"<<QThread::currentThreadId();
	xxwImgOp.fManualThreshold(src, dst, 50, 150);
	qDebug() <<"************" << QTime::currentTime() <<"**********************"<< QThread::currentThreadId();
 
}
 
int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);
    QFuture<void> f1 = run(hello,  src);
    QFuture<void> f2 = run(hello, src);
    //阻塞调用，阻塞主线程直到计算完成
    f1.waitForFinished();
    f2.waitForFinished();
 
    //阻塞为End的执行顺序
    qDebug() << "End";
    return a.exec();
}
```

三、特点：

//调用外部函数 QFuture<void> f1 =QtConcurrent::run(func,QString("aaa"));

//调用类成员函数 QFuture<void> f2 =QtConcurrent::run(this,&MainWindow::myFunc,QString("bbb"));

要为其指定线程池，可以将线程池的指针作为第一个参数传递进去

向该函数传递参数，需要传递的参数，则跟在函数名之后

可以用run函数的返回值funIr来控制线程。
 如： funIr.waitForFinished(); 等待线程结束，实现阻塞。
funIr.isFinished()  判断线程是否结束
funIr, isRunning()   判断线程是否在运行
funIr的类型必须和线程函数的返回值类型相同，可以通过
funIr.result()  取出线程函数的返回值

 缺点，不能直接用信号和槽函数来操作线程函数，eg : 当线程函数结束时，不会触发任何信号。