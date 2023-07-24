Qt connect 第五个参数

 一，Qt connect 函数原型如下，第五个（5种）参数根据接收者和发送者是否在同一个线程不同

QObject::connect(const QObject *sender, const char *signal, const QObject *receiver, const char *method, Qt::ConnectionType type = Qt::AutoConnection)
二，Qt::ConnectionType  详解

 

Qt::AutoConnection 
默认值，使用此值，连接类型会在信号发射时决定。如果接收者和发送者在同一个线程，则自动使用Qt::DirectConnection类型，如果接收者和发送者不在同一个线程，则自动使用Qt::QueuedConnection类型。

 

Qt::DirectConnection
槽函数运行于信号发送者所在的线程，效果上就像是直接在信号发送的位置调用了槽函数。多线程下比较危险，可能会造成崩溃。

 

Qt::QueuedConnection
槽函数在控制回到接收者所在线程的事件循环时被调用，槽函数运行于信号接收者所在线程。发送信号后，槽函数不会立即被调用，等到接收者当前函数执行完，进入事件循环之后，槽函数才会被调用。多线程下用这个类型。

 

Qt::BlockingQueuedConnection
槽函数的调用时机与Qt::QueuedConnection 一致，不过在发送完信号后，发送者所在线程会阻塞，直到槽函数运行完。接收者和发送者绝对不能在一个线程，否则会死锁。在多线程间需要同步的场合会用到这个。

 

Qt::UniqueConnection
此类型可通过 “|”  与以上四个结合在一起使用。此类型为当某个信号和槽已经连接时，在进行重复连接时就会失败，可避免重复连接。如果重复连接，槽函数会重复执行。




三， lambda表达式方式 连接

如果槽函数很简单，可以直接利用 lambda表达式进行连接，以减少代码量

//这里需要注意 Lambda表达式是C++ 11 的内容，所以，需要再Pro项目文件中加入 CONFIG += C++ 11
    QObject::connect(ui->pushButton,&QPushButton::clicked,[=](){qDebug()<<"lambda 表达式";});