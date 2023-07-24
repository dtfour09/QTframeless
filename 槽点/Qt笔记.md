实现隐藏窗口栏。

```QT
this->setWindowFlags(Qt::FramelessWindowHint);
```

关于error   redefinition of '***'

```
#ifndef _MAP_H_
#define _MAP_H_
 
//头文件内容插入里面　
class map{
};
 
#endif  
/*重复调用，加入这个就行了*/
```

Qt获得当前窗口的大小

```
this.width    this.height
```

QT设置窗口大小

```
setFixedHeight     setFixedWidth
```

QT隐藏窗口和显示窗口

```
this.hide   this.show
```

QBoxLayout可以在水平方向或垂直方向上排列控件，由QHBoxLayout、QVBoxLayout所继承。



关于无法捕捉鼠标移动事件

```
*在Qt中要捕捉鼠标移动事件需要重写MouseMoveEvent，但是MouseMoveEvent为了不太耗资源，默认状态下是要鼠标按下才能捕捉到。要想鼠标不按下时的移动也能捕捉到，需要setMouseTracking(true)。
QWidget中使用是没有问题的，但是，对于QMainWindow即使使用了setMouseTracking(true)依然无法捕捉到鼠标没有按下的移动，只有在鼠标按下是才能捕捉。
解决办法：要先把QMainWindow的CentrolWidget使用setMouseTracking(true)开启移动监视。然后在把QMainWindow的setMouseTracking(true)开启监视。之后就一切正常了。
原因：CentrolWIdget是QMainWindow的子类，你如果在子类上响应鼠标事件，只会触发子类的mouseMoveEvent，根据C++继承和重载的原理，所以子类也要setMouseTracking(true); 所以如果你想响应鼠标事件的控件被某个父控件包含，则该控件及其父控件或容器也需要setMouseTracking(true);*
```

Qt图片自适应窗口控件大小（使用setScaledContents）

QT的子窗口要浮在主窗口的上边，要用raise（）。

可以这样利用是得头文件不会重复

```
    
    IsFather1 的类型是（QWidget）
    IsFather1 = parent;
    MainWindow *IsFather = (MainWindow * ) IsFather1;
```

QT获得嵌入窗口在上一级的相对位置

```
OldWindowsRect = IsFather->geometry();
```

获得鼠标的在屏幕上的绝对坐标

```
QCursor::pos()
获得相对坐标是QMouseEvent->pos()
```

```
受保护的成员在派生类中可以使用
而私有的不可以
```

QT的QTimer的临时运用

```
QTimer::singleShot()
```

QT的鼠标窗口检查 move的

```
this->setMouseTracking(true); 
```

QT  QPainter绘画事件报错

```
QBackingStore::endPaint() called with active painter; did you forget to destroy it or call QPainter::end() on it? QPaintDevice: Cannot destroy paint device that is being painted
解决方案： QPainter pa(this);
```

QT 信号的问题

这个问题不太好解决

```
要单独的中间加一个类来写信号  QObject 类   注意信号只能写在继承这个函数的内部
```

QTtimer每过一段时间响应一件事情

```
    QTimer *timer = new QTimer(widget);
    timer->setInterval(5000); // 5000毫秒
    timer->start();

    // 连接定时器的timeout()信号到槽函数
    QObject::connect(timer, &QTimer::timeout, [=]() {
        QMessageBox::information(widget, "Message", "Timer timeout!");
    });
```

QT 更改样式

```
void CMainTitleBar::ClearLayout()
{
    while(m_pMainLayout->count())
    {
        QWidget *pWidget=m_pMainLayout->itemAt(0)->widget();
        if (pWidget)
        {
            pWidget->setParent (NULL);
            m_pMainLayout->removeWidget(pWidget);
            delete pWidget;
        }
        else
        {
            QLayout *pLayout=m_pMainLayout->itemAt(0)->layout();
            if (pLayout)
            {
                while(pLayout->count())
                {
                    QWidget *pTempWidget=pLayout->itemAt(0)->widget();
                    if (pTempWidget)
                    {
                        pTempWidget->setParent (NULL);

                        pLayout->removeWidget(pTempWidget);
                        delete pTempWidget;
                    }
                    else
                    {
                        pLayout->removeItem(pLayout->itemAt(0));
                    }
                }
            }
            m_pMainLayout->removeItem(m_pMainLayout->itemAt(0));
        }
    }
}
```

setFixedsize（）

```
巨坑  设置后没办法改变大小、
```

```
addStretch();//添加一个可伸缩空间

addSpacing(int size);//添加一个固定size 大小的间距

setMargin(int);//setMargin可以设置左、上、右、下的外边距，设置之后，他们的外边距是相同的

//与setMargin功能相同，但是可以将左、上、右、下的外边距设置为不同的值
setContentsMargins(int left, int top, int right, int bottom );

setContentsMargins(const QMargins &margins); 设置外边距

addWidget(QWidget *, int stretch = 0, Qt::Alignment alignment = 0) //添加控件
默认的，我们添加控件至水平布局中，默认都是垂直方向居中对齐的。

setDirection(QBoxLayout::RightToLeft)//设置布局方向

setStretchFactor(QWidget *w, int stretch);//设置控件、布局的拉伸系数
setStretchFactor(QLayout *l, int stretch); 

```

关于乌班图下的一些问题。

```
他的全屏未改变实际窗口大小 ， 不知道为什么。
响应有问题也不知道为啥。
f_bar  不设置为中心窗口会导致失控，应该是失控，另外raise失效  要使用m_pFloatbar->setWindowFlag(Qt::WindowStaysOnTopHint);
另外布局有问题，布局在ui上很好用，但是只是代码的话就很难
dpi超过 200 不算两百失效问题。
已经做好的，动态换窗口。
```

关于mac os的一些问题

![img](https://img-blog.csdnimg.cn/img_convert/1f5d6e092fc75f4c1319460cbc2555f5.png)

```
mac上面底部的statusBar没有关闭
Qt的按钮都是使用的系统按钮所以在macos上需要设定自己的样式。
```

```
public/private/signals/protected/slots 前面加一行空行隔开，按照public/signals/slots/protected/private的顺序定义
```

三种窗口置顶方式

```
具体操作
一、针对第一种锁定弹出窗口
1、如果窗口是基于QDialog创建。
topWindow.setParent(this);//指定父窗口，一般是目前将你弹出的窗口
topWindow.exec();//模态
2、如果窗口是基于QWidget创建（不建议这么做）
	topWindow->setWindowFlags(topWindow->windowFlags() |Qt::Dialog);
	topWindow->setWindowModality(Qt::ApplicationModal); //阻塞除当前窗体之外的所有的窗体
	topWindow->show();
二、设置窗口一直保持在顶层，但是不阻塞用户操作其他窗口
1、如果窗口是基于QDialog创建的
topWindow=new TopWindow(this);//指定父窗口
topWindow.show();//非模态
2、如果窗口是基于QWidget创建的(不建议)
topWindow.setParent(this);//指定父窗口
topWindow.setWindowFlags(topWindow.windowflags()| Qt::Dialog); //乌班图上要加入这个，目前还不太清楚为什么
topWindow.show();
以上topWindow是窗体类新建的对象。
其他的窗口置顶方式
其实有其他将窗口置顶的方式，但是是对于所有程序的窗口都置顶。也就是说其他程序打开后也在被置顶的窗口所遮盖。
topWindow->setWindowFlags(topWindow->windowFlags() | Qt::WindowStaysOnTopHint);
或者
SetWindowPos(HWND(topWindow->winid()),HWND_TOPMOST,0,0,0,0,SWP_NOMOVE|SWP_NOSIZE);
topWindow是要设置置顶的窗口。
```

关于setStyleSheet失效的问题。

```
原因是应为样式表问题。
每个窗口需要设置自己的样式名字。
如setObjectName("titleBarButton");
然后再样式命名的时候。
加入样式的名称
QString("#titleBarButton "
                          "{background-color: transparent;"
                          "border-radius: %1px;}").arg(radius_size);
                          这样才能精确的区修改样式。
```

关于图片不显示问题。

```
Qt的运行目录需要设置下。
```

