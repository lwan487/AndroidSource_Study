[TOC]
&#8195;&#8195;我们知道，Android应用程序是通过消息来驱动的。Android应用程序的每一个线程在启动时，都可以首先在内部创建一个消息队列，然后在进入到一个无限循环中，不断检查他的消息队列是否有新的消息需要处理。如果有新的消息需要处理，那么线程就会将它从消息队列中取出来，并且对它进行处理；否则，线程就进入睡眠等待状态，知道有新的消息需要处理为止。这样可以通过消息来驱动Android应用程序的执行了。

&#8195;&#8195;我们将线程的生命周期划分为创建消息队列和进入消息循环两个阶段，其中，消息循环阶段又可以划分为发送消息和处理消息两个子阶段，它们是交替进行的。

&#8195;&#8195;Android系统主要通过MessageQueue、Looper和Handler三个类来实现Android应用程序的消息处理机制，其功能如下去所示：

![image](https://note.youdao.com/yws/api/personal/file/8747800512464178A7A39187518489F7?method=download&shareKey=efa3303bf05b329bec815ee7cf9018be)

## 创建消息队列
&#8195;&#8195;我们通过调用Looper类的静态成员函数prepareMainLooper或者prepare来创建MessageQueue，其中，前者用来为应用程序的主线程创建消息队列；后者用来为应用程序的其他字线程创建消息队列。

&#8195;&#8195;Android应用程序的消息处理机制不仅可以在Java代码中使用，也可以在C++ 代码中使用，并且Java层中的Looper类和MessageQueue类是通过C++层中的Looper类和NativeMessageQueue类来实现的，它们的关系如下图：

![image](https://note.youdao.com/yws/api/personal/file/F1906AC095E6401DAD209E2BF8E7FF17?method=download&shareKey=a1905c336d55bba8dba50399220bdf22)

&#8195;&#8195;Java层中的每一个MessageQueue对象有一个类型为int的成员变量mPtr，它保存了 C++ 层中的一个NativeMessageQueue对象的地址值，这样就可以将Java层中的一个MessageQueue对象与C++层中的一个NativeMessageQueue对象关联起来。

&#8195;&#8195;Java层中的每一个MessageQueue对象还有一个类型为Message的成员变量mMessages，用来描述一个消息队列，可以调用MessageQueue类的成员函数enqueueMessage来往里面添加一个消息。

&#8195;&#8195;C++层中的Looper对象有一个类型为int的成员变量**mWakeEventFd**的文件描述符，Linux的event机制在发现mWakeEventFd文件描述符上有事件发生时，会从Looper类的pollInner方法中的epoll_wait唤醒，继续处理消息。

&#8195;&#8195;当调用Java层的Looper类的静态成员函数prepareMainLooper或者prepare来为一个线程创建一个消息队列时，Java层的Looper类就会在这个线程中创建一个Looper对象和一个MessageQueue对象。在创建Java层的MessageQueue对象的过程中，又会调用自身成员变量nativeInit在C++ 层中创建一个NativeMessageQueue对象和一个Looper对象。在创建C++层的Looper对象时，创建文件描述符mWakeEventFd。

&#8195;&#8195;Java层中的Looper类的成员函数loop、MessageQueue类的成员函数nativePollOnce和nativeWake，以及C++层中的NativeMessageQueue类的成员函数pollOnce和wake、Looper类的成员函数pollOnce和wake，是与线程的消息循环、消息发送和消息处理相关的。

&#8195;&#8195;接下来我们分析一下Android应用程序线程的消息队列的创建过程，即分析Looper类的静态成员函数prepareMainLooper和prepare的实现：

&#8195;&#8195;***frameworks/base/core/java/android/os/Looper.java***
```java
package android.os;

import android.annotation.NonNull;
import android.annotation.Nullable;
import android.os.LooperProto;
import android.util.Log;
import android.util.Printer;
import android.util.Slog;
import android.util.proto.ProtoOutputStream;

public final class Looper {
    ......

    // 用于保存线程中的Looper对象，可以理解为一个线程局部变量或者一个HashMap
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static Looper sMainLooper;  // guarded by Looper.class

    final MessageQueue mQueue;
    final Thread mThread;

    ......
    
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        // 检查当前线程是否已经有一个Looper对象。如果有，那么代码抛出异常，否则就会创建一个Looper对象
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    public static void prepareMainLooper() {
       // 调用Looper类的静态成员函数prepare在当前线程中创建一个Looper对象
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            // 调用Looper的静态成员函数myLooper获取Looper并将其保存在Looper的静态成员变量mMainLooper中。
            sMainLooper = myLooper();
        }
    }

    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

    ......
    
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

    ......
}

```

> 注意<html><br/></html>
>&#8195;&#8195;Looper类的静态成员函数prepareMainLooper只能在Android应用程序主线程中调用。Android应用程序主线程是一个特殊的线程，因为只有它才可以执行与UI相关的操作，因此，我们又将它称为UI线程。将Android应用程序主线程的Looper对象另外保存在一个独立的静态成员变量中，是为了让其他线程可以通过Looper类的静态成员函数getMainLooper来访问它，从而可以往它的消息队列中发送一些与UI操作相关的消息。

&#8195;&#8195;一个Looper对象在创建的过程中，会在内部创建一个MessageQueue对象，并且保存在它的成员变量mQueue中，如下所示。

&#8195;&#8195;***frameworks/base/core/java/android/os/Looper.java***
```java
package android.os;

import android.annotation.NonNull;
import android.annotation.Nullable;
import android.os.LooperProto;
import android.util.Log;
import android.util.Printer;
import android.util.Slog;
import android.util.proto.ProtoOutputStream;

public final class Looper {
    ......

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

    ......
}

```

&#8195;&#8195;一个MessageQueue对象在创建过程中，又会在C++层创建一个NativeMessageQueue对象，这是通过调用MessageQueue类的成员函数nativeInit来实现的，如下所示：

&#8195;&#8195;***frameworks/base/core/java/android/os/MessageQueue.java***
```java
package android.os;

import android.annotation.IntDef;
import android.annotation.NonNull;
import android.os.MessageQueueProto;
import android.util.Log;
import android.util.Printer;
import android.util.SparseArray;
import android.util.proto.ProtoOutputStream;

import java.io.FileDescriptor;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.util.ArrayList;

public final class MessageQueue {
    ......
    
    private final boolan mQuitAllowed;

    private long mPtr; // used by native code
    
    private native void nativeInit();
    
    MessageQueue(boolean quitAllowed){
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
    ......
}
```

&#8195;&#8195;MessageQueue类的成员函数nativeInit是一个JNI方法，它是由C++层中的函数android_os_MessageQueue_nativeInit来实现，如下所示。

&#8195;&#8195;***frameworks/base/core/jni/android_os_MessageQueue.cpp***
```C++
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz){
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env,"Unable to allocate native queue");
        return 0;
    }
    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

&#8195;&#8195;gMessageQueueClassInfo是一个全局变量，它的定义如下所示。

&#8195;&#8195;***frameworks/base/core/jni/android_os_MessageQueue.cpp***
```c++
static struct {
    jfieldID mPtr; // native object attached to the DVM MessageQueue
    jmethodID dispatchEvents;
} gMessageQueueClassInfo;
```

&#8195;&#8195;它的成员变量mPtr指向了Java层的MessageQueue类的mPtr属性，成员变量dispatchEvents指定了MessageQueue类的成员函数<html><code>private int dispatchEvent(int fd, events)</code></html>；

&#8195;&#8195;从全局变量gMessageQueueClassInfo的定义就可以看出，函数android_os_MessageQueue_nativeInit实际上是将一个C++层的NativeMessageQueue对象的指针保存在Java层的MessageQueue对象的成员变量mPtr中，从而可以将他们关联起来。

&#8195;&#8195;一个NativeMessageQueue对象在创建的过程中，又会在内部创建一个C++层的Looper对象，如下所示。

&#8195;&#8195;***frameworks/base/core/jni/android_os_MessageQueue.cpp***
```c++
1 NativeMessageQueue:NativeMessageQueue():mPollEnv(NULL),
2     mPollObj(NULL),mExceptionObj(NULL) 
3 {
4     mLooper = Looper::getForThread();
5     if (mLooper == NULL) {
6         mLooper = new Looper(false);
7         Looper::setForThread(mLooper);
8     }
9 }
```

&#8195;&#8195;第4行代码首先调用C++ 层的Looper类的静态成员函数getForThread来检查是否已经为当前线程创建过一个C++ 层的Looper对象。如果没有创建过，那么接着第6行代码首先创建一个C++ 层的Looper对象，并且将它保存在NativeMessageQueue类的成员变量mLooper中，然后在第7行代码再调用C++ 层的Looper类的静态成员函数setForThread将当前线程关联起来。

&#8195;&#8195;一个C++ 层的Looper对象在创建的过程中，又会在内部创建一个管道，如下所示。

&#8195;&#8195;***system/core/libutils/Looper.cpp***
```C++
01 Looper::Looper(bool allowNonCallbacks):
02        mAllowNonCallbacks(allowNonCallbacks),
03        mSendingMessage(false),
04        mPolling(false),
05        mEpollFd(-1),
06        mEpollRebuildRequired(false),
07        mNextRequestSeq(0),
08        mResponseIndex(0),
09        mNextMessageUptime(LLONG_MAX) {
10     mWakeEventFd = eventfd(0, EFD_NONBLOCK);
11     LOG_ALWAYS_FATAL_IF(mWakeEventFd < 0, "Could not make  wake event fd. errno=%", errno);

12     AutoMutex _l(mLock);
13     rebuildEpollLocked();
14 }
15 
16 void Looper::rebuildEpollLocked() {
17     // close old epoll instance if we have one.
18     if (mEpollFd >= 0) {
19        close(mEpollFd);
20     }
21 
22     // Allocate the new epoll instance and register the wake pip.
23     mEpollFd = epoll_create(EPOLL_SIZE_HINT);
24     
25     struct epoll_event eventItem;
26     memset(&eventItem, 0, sizeof(epoll_event)); // zero out unused memebers of data field union
27     eventItem.events = EPOLLIN;
28     eventItem.data.fd = mWakeEventFd;
29     int result = epoll_ctl(mEpllFd, EPOLL_CTL_ADD, mWakeEventFd, &eventITem);
30     
31     for (size_t i = 0; i < mRequests.size(); i++) {
32         const Request& request = mRequests.valueAt(i);
33         struct epoll_event eventItem;
34         reqeust.initEventItem(&eventITem);
35
36         int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, &eventItem);
37         if(epollResult < 0) {
38             ALOGE("Error adding epoll events for fd %d while rebuilding epoll set, errno=%", request.fd, errno);
39         }
40     }
41 }
```

&#8195;&#8195;第10行，创建eventfd并赋值给mWakeEventFd，在以前的Android版本上，这里创建的是pipe管道。evnetfd是较新的API，被用作一个事件等待/响应，实现了线程之间事件通知。

&#8195;&#8195;rebuildEpollLocked中通过epoll_create创建了一个epoll专用的文件描述符，EPOLL_SIZE_HINT表示mEpollFd上能监控的最大文件描述符。最后调用epoll_ctl监控mWakeEventFd文件描述符的EPOLLIN事件，即当eventfd中有内容可读时，就唤醒当前正在等待的线程。

&#8195;&#8195;Linux系统的epoll机制是为了同时监听多个文件描述的IO读写事件而设计的，它是一个多路复用IO接口，类似于Linux系统的select机制，但是它是select机制的增强版。如果一个epoll实例监听了大量的文件描述符的IO读写事件，但是只有少量的文件描述符是活跃的，即只有少量的文件描述符会发生IO读写事件，那么这个epoll实例可以显著地减少CPU的使用率，从而提供系统的并发处理能力。

## 线程消息循环过程
&#8195;&#8195;一个Android应用程序线程的消息队列创建完成之后，就可以调用Looper类的静态成员函数loop使它进入到一个消息循环中，如下图所示。

![image](https://note.youdao.com/yws/api/personal/file/B32CD41A6A7B4126A157DC31A3174FC6?method=download&shareKey=62570b83945e81551ee5113f095cc04a)

&#8195;&#8195;这个过程一共分为7个步骤，下面我们来分析一下每一个步骤。

**Step 1: Lopper.loop**
&#8195;&#8195;***frameworks/base/core/java/android/os/Looper.java***
```java
01 public class Looper {
02     ......
03     
04     public static final void loop() {
05         final Looper me = myLooper();
06        if (me == null) {
07             throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.")
08         }
09         final MessageQueue queue = me.mQueue;
10        ......
11        for (;;) {
12            Message msg = queue.next(); // might block
13            if (msg == null) {
14                // No message indicates that the message queue is quitting.
15                return;
16            }
17            ......
18        }
19     }
20    
21     ......
22 }
```
&#8195;&#8195;第5行到第9行代码首先获得当前线程的消息队列，接着第11行到第18行的for循环不断地检查这个消息队列中是否有新的消息需要处理。如果有新的消息需要处理，那么第12行代码获得的Message对象msg就不等于null；否则当前线程就会在MessageQueue类的成员函数next中进入睡眠等待状态，直到有新的消息需要处理为止。

**Step 2：MessageQueue.next**
&#8195;&#8195;***frameworks/base/core/java/android/os/MessageQueue.java***

```java
01 public class MessageQueue {
02     ......
03    
04     Message next() {
05        // Return here if the message loop has already quit and been disposed
06        // This can happen if the application tries to restart a looper after quit
07        // which is not supported
08        final long ptr = mPtr;
09        if (ptr == 0) {
10            return null;
11        }
12         
13        int pendingIdleHandlerCount = -1; // -1 only during first iteration
14        int nextPollTimeoutMillis = 0;
15         
16        for (;;) {
17            if (nextPollTimeoutMillis != 0) {
18                 Binder.flushPendingCommands();
19            }
20            
21             nativePollOnce(ptr, nextPollTimeoutMillis);
22            
23             synchronized(this) {
24                 // Try to retrieve  the next message. Return if found.
25                 final long now = SystemClock.uptimeMillis();
26                 Message prevMsg = null;
27                 Message msg = mMessages;
28                 if (msg != null && msg.target == null) {
29                     // Started by a barrier. Find the next asynchronous message in queue.
30                     do {
31                         prevMsg = msg;
32                         msg = msg.next;
33                     } while (msg != null && !msg.isAsynchronous());
34                 }
35                 if (msg != null) {
36                     if (now < msg.when) {
37                         // Next message is not ready.Set a timeout to wake up when it is ready.
38                         nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.Max_VALUE);
39                     } else {
40                         // Got a message
41                         mBlocked = false;
42                         if (prevMsg != null) {
43                             prevMsg.next = msg.next;
44                         } else {
45                             mMessages = msg.next;
46                         }
47                         msg.next = null;
48                         if (DEBUG) Log.v(TAG, "Returning message:" + msg);
49                         msg.markInUse();
50                         return msg;
51                     }
52                 } else {
53                     // No more messages.
54                     nextPollTimeoutMillis = -1;
55                 }
56                 
57                 if (mQuitting) {
58                     dispose();
59                     return null;
60                 }
61                
63                 ......
64                 // Get the idle handlers
65                 ......
66             }
67            
68             // Run the idle handlers.
69             ......

70             // Reset the idle handler count to 0 so we do not run them again.
71             pendingIdleHandlerCount = 0;
        
72             // While calling an idle handler, a new message could have been delivered
73             // so go back and look again for a pending message without waiting.
74             nextPollTimeoutMillis = 0;
75         }
76     }

77     ......
78 }
```
&#8195;&#8195;第13行代码定义了一个变量pendingIdleHandlerCount,用来保存注册到消息队列中的空闲消息处理器（IdleHandler）的个数，它们是在第64行省略的代码中计算的。当线程发现它的消息队列没有新的消息需要处理时，不是马上进入睡眠等待状态，而是调用注册到它的消息队列中的IdleHandler对象的成员函数queueIdle，以便有机会在线程空闲时执行一些操作。这些IdleHandler对象的成员函数queueIdle在第69行省略的代码中调用，在后面的章节中，我们分析。

&#8195;&#8195;第14行代码定义了另外一个变量nextPollTimeoutMillis。用来描述当消息队列中没有新的消息需要处理时，当前线程需要进入睡眠等待状态的时间。如果变量nextPollTimeoutMillis的值等于0，那么就表示即使消息队列中没有新的消息需要处理，当前线程需要进入睡眠等待状态的时间。如果nextPollTimeoutMillis的值等于0，那么就表示即使消息队列中没有新的消息需要处理，当前线程也不要进入睡眠等待状态。如果变量nextPollTimeoutMillis的值等于-1，那么就表示当消息没有新的消息需要处理时，当前线程需要无限地处理睡眠等待状态，直到它被其他线程唤醒为止。

&#8195;&#8195;第16行到第75行的for循环不断地调用成员函数nativePollOnce来检查当前线程的消息队列中是否有新的消息需要处理。

```java
注意：
在调用成员函数nativePollOnce时，当前线程可能会进入睡眠等待状态，那么进入睡眠等待状态的时间就由第二个参数nextPollTimeoutMillis来指定。
```

&#8195;&#8195;MessageQueue类内部有一个类型为Mesage的成员变量mMesages，用来描述当前线程需要处理的消息。当前线程从成员函数nativePollOnce返回来之后，如果它有新的消息需要处理，即MessageQueue类的成员变量mMessages不等于null，那么接下来第36行到56行代码就会对它进行处理，否则，第54行代码就会将变量nextPollTimeoutMillis的值设置为-1，表示当前线程下次在调用成员函数nativePollOnce时，如果没有新的消息需要处理，那么就要无限地处于睡眠等待状态，直到它被其他线程唤醒为止。

&#8195;&#8195;第36行代码比较MessageQueue类的成员变量mMessages所描述的消息的处理时间大于系统的当前时间，那么说明当前线程不需要马上对它进行处理。因此，第38行代码会计算当前线程下一次调用成员函数nativePollOnce时，如果没有新的消息需要处理，那么当前线程需要进入睡眠等待状态的时间，以便可以在指定的时间点对接下来的消息进行处理。否则，第40行到51行代码就会将它返回给上一步处理，并且将mMessageQueue类的成员变量mMessages重新指向下一个需要处理的消息。

&#8195;&#8195;当前线程执行到第69行代码时，就说明它目前没有新的消息需要处理，于是接下来它就会重新执行第16行到75行的for循环等待下一个需要处理的消息。这时候当前线程需要进入睡眠等待状态的时间保存在变量nextPollTimeoutMillis中。不过，在进入睡眠等待状态前，当前线程会分发一个线程空闲消息给那些已经注册到消息队列中的IdleHandler对象处理，如69行省略的代码所示。

&#8195;&#8195;第74行代码是在注册到消息队列中的IdleHandler对象处理完成了一个线程空闲消息后执行的，它主要是将nextPollTimeoutMillis的值重新设置为0，表示当前线程接下来在调用成员函数nativePollOnce时，不可以进入睡眠等待状态。由于注册的IdleHandler对象在处理线程空闲消息期间，其他线程可能已经向当前线程的消息队列发了一个或者若干个消息，因此，这时候当前线程就不能够在成员函数nativePollOnce中进入睡眠等待状态，而是要马上返回来检查它的消息队列中是否有新的消息需要处理。

```java
第16行到75行的for循环中，每一次调用成员函数nativePollOnce之前，第9行的if语句都会检查变量nextPollTimemoutMillis的值是否等于0。如果不等于0，那么就说明当前线程会在成员函数nativePollOnce中进入睡眠等待状态，这时候低18行代码就会调用Binder类的静态成员函数fushPendingCommands来处理那些正在等待处理的Binder进程间信请求，避免它们长时间得不到处理。
```

**Step 3: MessageQueue.nativePollOnce**
```java
frameworks/base/core/jni/android_os_MessageQueue.cpp
1 static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jclass clazz, jlong ptr) {
2     NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
3     nativeMessageQueue->pollONce(env, obj, timeoutMillis);
4 }
```
&#8195;&#8195;参数obj指向了一个Java层的MessageQueue对象，而参数ptr指向了这个MessageQueue对象的成员变量mPtr。从前面的内容可以知道，MessageQueue类的成员变量mPtr保存的是一个C++层的NativeMessageQueue对象地址。因此，第3行代码就可以安全地将参数ptr转换成一个NativeMessageQueue对象，接着第4行代码再调用这个NativeMessageQueue对象的成员函数pollOnce来检查当前线程是否有新的消息需要处理。

**Step 4: NativeMessageQueue.pollOnce**
```java
frameworks/base/core/jni/android_os_MessageQueue.cpp
01 void NativeMessageQueue:pollOnce(JNIEnv* env, jobject pollOjb, int timeoutMillis) {
02     mPollEnv = env;
03     mPollObj = pollObj;
04     mLooper->pollOnce(timeoutMills);
05     mPollObj = NULL;
06     mPollEnv = NULL;
    
07     if (mExceptionObj) {
08         env->Throw(mExceptionObj);
09         env->DeleteLocalRef(mExceptionObj);
10         mExceptionObj = NULL:
11     }
12 }
```

&#8195;&#8195;从前面的内容可以知道，NativeMessageQueue类的成员变量mLooper指向一个C++层的Looper对象，第4行代码调用它的成员函数pollOnce来检查当前线程是否有新的消息需要处理。

**Step 5:Looper.pollOnce**
```java
system/core/libutils/Looper.cpp
01 int Looper:pollOnce(int timeoutMillis, int* outFd, int * outEvents, void** outData) {
02     int result = 0;
03     for (;;) {
04         ......
05        
06         if (result != 0) {
07             ......
08            
09             return result;
10         }
11
12         result = pollInner(timeoutMillis);
13     }
14 }
```

&#8195;&#8195;第3行到第13行的for循环不断地调用成员函数pollInner来检查当前线程是否有新的消息需要处理。如果有新的消息需要处理，那么成员函数pollInner的返回值就不会等于0，这时候第9行代码就会跳出第3行到第13行的for循环，以便当前线程可以对新的消息进行处理。

**Step 6: Looper:pollInner**
```java
system/core/libutils/Looper.cpp
01 int Looper:pollInner(int timeoutMills) {
02     ......
03    
04     int result = POLL_WAKE;
05     ......
06     
07     struct epoll_event eventItems[EPOLL_MAX_EVENTS];
08     int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
09     ......
10     
11     for (int i = 0; i < eventCount; i++) {
12         int fd = eventItems[i].data.fd;
13         uint32_t epollEvents = eventItems[i].events;
14         if (fd == mWakeEventFd) {
15             if (epollEvents & EPOLLIN) {
16                 awoken();
17             }
18             ......
19         }
20         ......
21     }
22     ......
23     
24     return result;
25 }
```

&#8195;&#8195;在前面的描述中，我们为当前线程在C++ 层创建一个epoll实例，并且将它的文件描述符保存在C++层的Looper类的成员变量mEpollFd中，同时mWakeEventFd加入到epoll监听队列中。

&#8195;&#8195;第8行代码调用函数epoll_wait来监听注册在前面所创建的epoll实例中的Event。如果mWakeEventFd没有发生Event事件，那么当前线程就会在函数epoll_wait中进入睡眠等待状态，等待的事件由最后的一个参数timeoutMillis来指定。

&#8195;&#8195;从函数epoll_wait返回来之后，接下来第11行到第21行的for循环就检查是哪一个Event发生了事件。第14行的if语句检查mWakeEventFd是否发生了Linux的Event事件。如果是，并且它发生了Linux的Event的类型为EPOLLIN，即第15行的if语句为true，那么这个时候就说明其他线程向与当前线程所关联的写入了一个新的数据。

&#8195;&#8195;第16行代码调用成员函数awoken将与当前线程所关联的一个关联的数据读出来，以便当前线程下一次可以重新在这个管道上等待其他线程向它的消息队列发送一个新的消息。

**Step 7: Looper.awoken**
```java
system/core/libutils/Looper.cpp
1 void Looper::awoken() {
2     ......
3     
4     uint64_t counter;
5     TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t));
6 }
```

&#8195;&#8195;第5行代码调用函数read将与当前关联的一个管道的数据读出来。从这里可以看出，当前线程根本不关心写入到与它所关联的管道的数据是什么，它只是简单地将这些数据读出来，以便可以清理这个管道中的旧数据。这样当前线程在下一次消息循环时，如果没有新的消息需要处理，那么它就可以通过监听这个管道的IO写事件进入睡眠等待状态，直到其他线程向它的消息队列发送了一个新的消息为止。

&#8195;&#8195;假设当前线程没有新的消息需要处理，那么它就会在前面Step 6中通过调用函数epoll_wait进入睡眠等待状态，直到其他线程向它的消息队列发送了一个新的消息为止。接下来，继续分析其他线程向当前线程的消息队列发送消息的过程。

## 线程消息发送过程

&#8195;&#8195;Android系统提供了一个Handler类，用来向一个线程的消息队列发送一个消息，它的实现下图：

![image](https://note.youdao.com/yws/api/personal/file/6C49AB0A33EE4935ADB6DA229CA6BDE9?method=download&shareKey=546c008c354b33536bcf4a66d4d445c9)

&#8195;&#8195;Handler类内部有mLooper和mQueue两个成员变量，它们分别指向一个Looper对象和一个MessageQueue对象。Handler类还有sendMessage和handleMessage两个成员函数，其中，成员函数sendMessage用来向成员变量mQueue所描述的一个消息队列发送一个消息；而成员函数handleMessage用来处理这个消息，并且它是在与成员变量mLooper所关联的线程中被调用的。

&#8195;&#8195;使用Handler类的默认构造函数来创建一个Handler对象，这时候它的成员变量mLooper和mQueue就会分别指向与当前线程所关联的一个Looper对象和一个MessageQueue对象，如下所示：
```java
01 frameworks/base/core/java/android/os/Handler.java
02 public class Handler {
03     ......
04     public Handler(){
05         mLooper = Looper.myLooper();
06         ......
07         
08         mQueue = mLooper.mQueue;
09         ......
10     }
11     
12     final MessageQueue mQueue;
13     final Looper mLooper;
14     ......
15 }
```

&#8195;&#8195;从Handler类的成员函数sendMessage开始，分析向一个线程的消息队列发送一个消息的过程，如下图所示：
![image](https://note.youdao.com/yws/api/personal/file/F5EEB128D345412B832A67099027643A?method=download&shareKey=6139f3dfeb2abf1bdddd76fbb0b28b79)

&#8195;&#8195;这个过程一共分为5个步骤，下面分析每一个步骤：

**Step 1: Handler.sendMessage**
```java
frameworks/base/core/java/android/os/Handler.java
01 public class handler {
02     ......
03     
04     public final boolean sendMessage(Message msg) {
05         return sendMessageDelayed(msg, 0);
06     }
07     
08     public final boolean sendMessageDelayed(Message msg, long delayMillis) {
09         if (delayMillis < 0) {
10             delayMillis = 0;
11         }
12         return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
13     }
14     
15     public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
16         MessageQueue queue = mQueue;
17         if (queue == null) {
18             RuntimeException e = new RuntimeException(this + " sendMessageAtTime() called with no mQueue");
19             Log.w("Looper", e.getMessage(), e);
20             return false;
21         }
22         return enqueueMessage(queue, msg, uptimeMillis);
23     }
24     
25     ......
26
27     private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
28          msg.target = this;
29          if (mAsynchronous) {
30               msg.setAsynchronous(true);
31          }
32          return queue.enqueueMessage(msg, uptimeMills);
33     }
34        
35     ......
36 }
```

**Step 2: MessageQueue.enqueueMessage**
```java
frameworks/base/core/java/android/os/MessageQueue.java
01 public class MessageQueue {
02     ......
03     
04     boolean enqueueMessage(Message msg, long when) {
05         ......
06         
07         synchronized(this) {
08             ......
09             
10             msg.markInUse();
11             msg.when = when;
12             Message p = mMessages;
13             if (p == null || when == 0 || when < p.when){
14                 // New head, wake up the event queue if blocked
15                 msg.next = p;
16                 mMessages = msg;
17                 neekWake = mBlocked;
18             } else {
19               // Inserted within the middle of the queue. Usually we don't have to wake
20                 // up the event queue unless there is a barrier at the head of the queue
21                 // and the message is the earliest asynchronous message in the queue.
22                 needWake = mBlocked && p.target == null && msg.isAsychronous();
23                 Message prev;
24                 for (;;) {
25                     prev = p;
26                     p = p.next;
27                     if (p == null || when < p.when) {
28                         break;
29                     }
30                     if (needWake && p.isAsynchronous()) {
31                         needWake = false;
32                     }
33                 }
34                 msg.next = p; // invariant: p == prev.next
35                 prev.next = msg;
36             }
37             
38             // We can assume mPtr != 0 because mQuitting is false.
39             if (needWake) {
40                 nativeWake(mPtr);
41             }
42         }
43     }
44     return true;
45 }
```

&#8195;&#8195;由于一个消息队列中的消息是按照它们的处理时间从小到大的顺序来排序的，因此，当我们将一个消息发送到一个消息队列时，需要先根据这个消息的处理时间找到它在目标消息队列的合适位置，然后再将它插入到目标消息队列中。

&#8195;&#8195;分四种情况来讨论如何将一个消息插入到一个目标消息队列中。
1. 目标消息队列是一个空队列。
2. 插入的消息的处理时间等于0。
3. 插入的消息的处理时间小于保存在目标消息队列头的消息的处理时间。
4. 插入的消息的处理时间大于等于保存在目标消息队列头的消息的处理时间。

&#8195;&#8195;对于前面三种情况来说，要插入的消息都需要保存在目标消息队列的头部，如第15行到第17行代码所示。

```
如果一个消息的处理时间等于0，那么就说明这是一个优先级最高的消息，因此我们需要将它保存在目标消息队列的头部，以便它可以优先得到处理。
```

&#8195;&#8195;对于最后一种情况来说，要插入的消息需要保存在目标消息队列中间的某一个位置上，如第19行到第35行代码所示，其中，这个位置是通过第24行到第33行的for循环找到的。

```
如果要插入的消息的处理时间与目标消息队列中的某一个消息的处理时间相同，那么后来插入的消息会被保存在后面，这样就可以保证先发送的消息可以先获得处理。
```
&#8195;&#8195;一个线程将一个消息插入到一个目标消息队列之后，可能需要将目标线程唤醒，这需要分两种情况来讨论。
1. 插入的消息在目标消息队列中间。
2. 插入的消息在目标消息队列头部。

&#8195;&#8195;对于第一种情况来说，由于保存在目标消息队列头部的消息没有发生变化，因此，当线程无论如何都不需要对目标线程执行唤醒操作，这时候第22行代码就会将变量needWake的值设置为false。

&#8195;&#8195;对于第二种情况来说，由于保存在目标消息队列头部的消息发生了变化，因此，当前线程就需要将目标线程唤醒，以便它可以对保存在目标消息队列头部的新消息进行处理。但是，如果这时候目标线程不是正处于睡眠等待状态，那么当前线程就不需要对它执行唤醒操作。当前正在处理的MessageQueue对象的成员变量mBlocked记录了目标线程是否正处于睡眠状态。如果它的值等于true，那么就表示目标线程正处于睡眠等待状态，这时候当前线程就需要将它唤醒。第17行代码将这个成员变量的值保存在变量needWake中，以便当前线程接下来可以决定是否需要将目标线程唤醒。

&#8195;&#8195;将要发送的消息插入到目标消息队列中之后，第39行的if语句就检查变量needWake的值是否等于true。如果等于，那么接下来第31行就会调用MessageQueue的成员函数nativeWake将目标线程唤醒。

&#8195;&#8195;假设目标线程正处于睡眠等待状态，接下来继续分析MessageQueue类的成员函数nativeWake的实现。

**Step 3: MessageQueue.nativeWake**
```C++
frameworks/base/core/jni/android_os_MessageQueue.cpp
01 static void android_os_MessageQueue_nativeWake(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
02     NativeMessageQueue* nativeMessageQueue = reinterpret_cat<NativeMessageQueue*>(ptr);
03     nativeMessageQueue->wake();
04 }
```

**Step 4: NativeMessageQueue.wake**
```C++
frameworks/base/core/jni/android_os_MessageQueue.cpp
01 void NativeMEssageQueue::wake(){
02     mLooper->wake();
03 }
```

&#8195;&#8195;NativeMessageQueue类的成员变量mLooper指向了一个C++层的Looper对象，第2行代码调用它的成员函数wake来唤醒目标线程，以便它可以处理它的消息队列中的新消息。

**Step 5: Looper.wake**
```C++
system/core/libutils/Looper.cpp
01 void Looper::wake() {
02     ......
03     
04     uint64_t inc = 1;
05     ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
06     if (nWrite != sizeof(uint64_t)) {
07         if (errno != EAGAIN) {
08             ALOGW("Could not write wake sigal, errno=%d", errno);
09         }
10     }
11 }
```

&#8195;&#8195;从前面的内容可以知道，C++层的Looper类的成员变量mWakeEVentFd是用来描述epoll的读写描述。第5行代码调用函数write向它写入一个整型1，即向它所描述的epoll写入一个新的消息，这时候目标线程就会因为这个管道发生了一个IO写事件而被唤醒。

&#8195;&#8195;至此，一个线程消息发送过程就分析完成了。接下来，我们继续分析一个线程消息的处理过程。

## 线程消息处理过程

&#8195;&#8195;从前面的小结的内容可以知道，当一个线程没有新的消息需要处理时，它会在C++层的Looper类的成员函数pollInner中进入睡眠等待状态，因此，当这个线程有新的消息需要处理时，它首先会在C++层的Looper的成员函数pollInner中被唤醒，然后沿着之前的调用路径一直返回到java层的Looper类的静态成员loop中，最后就可以对新的消息进行处理了。

&#8195;&#8195;接下来，我们从Java层的Looper类的静态成员函数loop开始，分析一下线程消息的处理过程，如下图所示：
![image](https://note.youdao.com/yws/api/personal/file/F9E1D63B44D5451FBE3ACE9BEF10B22A?method=download&shareKey=6535953986f2acdbf9d2b83d1fda637f)

&#8195;&#8195;这个过程一共分为3个步骤，下面就详细分析每一个步骤。

**Step 1: Looper.loop**
```java
frameworks/base/core/java/android/os/Looper.java
01 public class Looper {
02     ......
03     
04     public static void loop() {
05         final Looper me = myLooper();
06         if (me == null) {
07             throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
08         }
09         final MessageQueue queue = me.mQueue;
10         
11         // Make sure the identity of this thread is that of the local process,
12         // and keep track of what that identity token actually is.
13         Binder.clearCallingIdentity();
14         final long ident = Binder.clearCallingIdentity();
15         
16         for (;;) {
17             Message msg = queue.next(); // might block
18             if (msg == null) {
19                 // No message indicates that the message queue is quitting.
20                 return;
21             }
22             
23             // This must be in a local variable, in case a UI event sets the logger
24             Printer logging = me.mLogging;
25             if (logging != null) {
26                 logging.println(">>>>> Dispatching to " + msg.target + " " + msg.callback + "：" + msg.what);
27             }
28             
29             msg.target.dispatchMessage(msg);
30             
31             if (logging != null) {
32                 logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
33             }
34             
35             // Make sure that during the course of dispatching the 
36             // identity of the thread wasn't corrupted.
37             final long newIdent = Binder.clearCallingIdentity();
38             if (ident != newIdent) {
39                 Log.wtf(TAG, "Thread identity changed from 0x" 
40                         + Long.toHexString(ident) + " to 0x" 
41                         + Long.toHexString(newIdent) + " while dispatching to " 
42                         + msg.target.getClass().getName() + " "
43                         + msg.callback() + " what=" + msg.what);
44             }
45             
46             msg.recycleUnchecked();
47         }
48     }
49 }
```

&#8195;&#8195;第17行代码将要处理的消息保存在Message对象msg中，接着第18行的if语句检查这个Message对象的msg是否等于null。如果等于null，那么就说明要处理的是一个退出消息，因此，第20行代码就会退出消息循环；否则第29行代码就将这个消息分发给Message对象msg的成员变量target来处理。

&#8195;&#8195;从前面可知，Message对象msg的成员变量target指向的是一个Handler对象，因此，接下来我们就继续分析Handler类的成员函数dispatchMessage的实现。

**Step 2：Handler.dispatchMessage**
```java
framework/base/core/java/android/os/Handler.java
01 public class Handler {
02     ......
03     
04     public void dispatchMessage(Message msg) {
05         if (msg.callback != null) {
06             handleCallback(msg);
07         } else {
08             if (mCallback != null) {
09                 if (mCallback.handleMessage(msg)) {
10                     return;
11                 }
12             }
13             handleMessage(msg);
14         }
15     }
16 }
```

&#8195;&#8195;Handler类的成员函数dispatchMessage按照以下顺序来分发一个消息。
1. 如果要处理的消息在发送时就指定了一个回调接口，即第5行的if语句为true，那么第6行代码就会调用Handler类的成员函数handleCallback来处理这个消息。
2. 如果条件1不满足，并且负责分发消息的Handler对象的成员变量mCallback指向一个回调接口，即第8行的if语句为true，那么第9行的if语句就会调用这个回调接口的成员函数handleMessage来处理这个消息。
3. 如果条件2不满足，或者负责分发消息的Handler对象的成员变量mCallback所指向的一个回调接口希望这个消息可以继续向下处理，即第9行的if语句为false，那么第13行代码就会调用Handler类的成员函数handleMessage来处理这个消息。

&#8195;&#8195;下面先分析前两种情况的消息处理过程，然后在Step 3中再分析第三种情况的消息处理过程。

&#8195;&#8195;Handler类的成员函数handleCallback的实现如下所示。
```java
frameworks/base/core/java/android/os/Handler.java
01 public class Handler {
02     ......
03     
04     private final void handleCallback(Message message) {
05         message.callback.run();
06     }
07     
08     ......
09 }
```
&#8195;&#8195;第5行代码调用消息对象message的成员变量callback的成员函数run来处理一个消息。Message类的成员变量callback指向的是一个Runnable对象，因此，第5行代码实际上是将一个消息交给了Runnable类的成员函数run来处理。

&#8195;&#8195;当调用Handler类的成员函数post来发送一个消息时，Handler类就会为这个消息置顶一个回调接口，如下所示：
```java
frameworks/base/core/java/android/os/Handler.java
01 public class Handler {
02     ......
03     
04     public final boolean post(Runnable r) {
05         return sendMessageDelayed(getPostMessage(r), 0);
06     }
07     
08     ......
09 }
```

&#8195;&#8195;参数r是一个类型为Runnable的对象，第6行代码在将它发送到一个消息队列之前，首先会调用Handler类的成员函数getPostManager将它分装成一个Message对象，这是因为消息队列只接收类型为Message的消息。

&#8195;&#8195;Handler类的成员函数getPostMessage的实现如下所示：
```java
frameworks/base/core/java/android/os/Handler.java
01 public class Handler {
02     ......
03     
04     private final Message getPostMessage(Runnable r) {
05         Message m = Message.obtain();
06         m.callback = r;
07         return m;
08     }
09     
10     ......
11 }
```

&#8195;&#8195;第5行代码首先创建了一个空的消息对象，接着第6行代码将这个消息对象的成员变量callback设置为参数r所描述的一个Runnable对象。这样我们就可以将一个Runnable对象当作一个消息发送到一个消息队列中去处理，这个消息最终分发给这个Runnable对象的成员函数run来处理。

&#8195;&#8195;Handler类在内部定义一个Callback接口，如下所示：
```java
frameworks/base/core/java/android/os/Handler.java
01 public class Handler {
02     ......
03     
04     public interface Callback {
05         public boolean handleMessage(Message msg);
06     }
07     
08     ......
09 }
```

&#8195;&#8195;这个接口只有一个成员函数handleMessage，它是用来处理一个消息的。

&#8195;&#8195;除了可以使用Handler类的默认构造函数来创建一个Handler对象外，我们还可以使用Handler类的另外一个构造函数来创建一个Handler对象，这个构建函数需要指定一个实现了Callback接口对象，如下所示：
```java
frameworks/base/core/java/android/os/Handler.java
01 public class Handler {
02     ......
03     
04     public Handler(Callback callback, boolean async) {
05         ......
06         
07         mLooper = Looper.myLooper();
08         ......
09         mQueue = mLooper.mQueue;
10         mCallback = callback;
11         mAsynchronous = async;
12     }
13     
14     ......
15 }
```

&#8195;&#8195;第7行和第9行代码除了会将与当前线程所关联的一个Looper对象和一个MessageQueue对象保存在Handler类的成员变量mLooper和mQueue中之外，第10行代码还会降参数callback所描述的一个Callback对象保存在Handler类的成员变量mCallback中。

&#8195;&#81956;这样就可以直接使用Handler类来发送消息，并且将这个消息交给一个实现了Callback接口的对象来处理。

&#8195;&#8195;以上就是前两种情况的消息处理过程，接下来继续分析第三情况的消息处理情况，这是通过调用Handler类的成员函数handleMessage来实现的。

**Step 3: Handler.handleMessage**
```java
frameworks/base/core/java/android/os/Handler.java
01 public class Handler {
02     ......
03     
04     /**
05      * Subclasses must implement this to receive messages.
06      */
07     public void handleMessage(Message msg) {
08     }
09
10     ......
11 }
```

&#8195;&#8195;线程消息都有一个共同的特点，即它们都是先被发送到一个消息队列中，然后再从这个消息队列中取出来处理的。有一种特殊消息，它们不用事先发送到一个消息队列中，而是由一个线程在空闲的时候主动发送出来，这种特殊的线程消息称为线程空闲消息，接下来分析它的处理过程。

&#8195;&#8195;线程空闲消息是由一种称为空闲消息处理器的对象来处理，这些空闲消息处理器必须要实现一个IdleHandler接口。

&#8195;&#8195;IdleHandler接口的定义如下定义：
```java
frameworks/base/core/java/android/os/MessageQueue.java
public class MessageQueue {
    ......
    
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more. Return true to keep your idle handler active, false
         * to have it removed. This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
         boolean queueIdle();
    }
    
    ......
}
```

&#8195;&#8195;它只有一个成员函数queueIdle，用来接收线程空闲消息。

&#8195;&#8195;一个空闲消息处理器如果想要接收到一个线程的空闲消息，那么它就必须要注册到这个线程的消息队列中，即注册到与这个线程所关联的一个MessageQueue对象中。

&#8195;&#8195;MessageQueue类有一个成员变量mIdleHandlers，它指向一个IdleHandler列表，用来保存空闲消息处理器，如下图所示：

![image](https://note.youdao.com/yws/api/personal/file/5844F6859CA9408A924B5F7803C69767?method=download&shareKey=930a37ef3ba542befff71d7cfe716b29)

&#8195;&#8195;MessageQueue类有两个成员函数addIdleHandler和remvoeIdleHandler，分别用来注册和注销一个空闲消息处理器，它们的实现如下所示：
```java
frameworks/base/core/java/android/os/MessageQueue.java
public class MessageQueue {
    ......
    
    public final void addIdleHandler(IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }
    
    public void removeIdleHandler(IdleHandler handler) {
        synchronized(this){
            mIdleHandlers.remove(handler);
        }
    }
    
    ......
}
```

&#8195;&#8195;从前面的小结中可以知道，一个线程在消息循环的过程中，有时候会变得“无事可做”。当一个线程的消息队列为空，或者保存在消息队列头部的消息的处理时间大于系统的当前时间时，就会发生这种情况，这时候线程就处于一种空闲状态，接下来它就会进入睡眠等待状态。不过，在进入睡眠等待状态之前，这个线程会发出一个线程空闲消息给那些注册了的空闲消息处理器来处理。

&#8195;&#8195;继续分析MessageQueue类的成员函数next的实现，以便可以了解线程空闲消息的处理过程。

&#8195;&#8195;在MessageQueue类的成员函数next中，与线程空闲消息的相关的代码如下所示。

```java
frameworks/base/core/java/android/os/MessageQueue.java
01 public class MessageQueue {
02     ......
03     
04     Message next() {
05         // Return here if the message loop has already quit and been disposed.
06         // THis can happen if the application tries to restart a looper after quit
07         // which is not supported
08         final long ptr = mPtr;
09         if (ptr == 0){
10             return null;
11         }
12         
13         int pendingIdleHandlerCount = -1; // -1 only during first iteration
14         ......
15         
16         for (;;) {
17             ......
18             nativePollOnce(mPtr, nextPollTimeoutMillis);
19             
20             synchronized(this) {
21                 ......
22                 
23                 // If first time idle, then get the number of idlers to run.
24                 // Idle handles only run if the queue is empty or if the first message
25                 // in the queue (possibly a barrier) is due to be handled in the future.
26                 if (pendingIdleHandlerCount < 0
27                         && (mMessage == null || now < mMesssage.when)) {
28                     pendingIdleHandlerCount = mIdleHandlers.size();        
29                 }
30                 if (pendingIdleHandlerCount <= 0){
31                     // No idle handlers to run. Loop and wait some more.
32                     mBlocked = true;
33                     continue;
34                 }
35                 if (mPendingIdleHandlers == null) {
36                     mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
37                 }
38                 mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
39             }
40             
41             // Run the idle handlers.
42             // We only ever reach this code block during the first iteration.
43             for (int i = 0; i < pendingIdleHandlerCount; i++) {
44                 final IdleHandler idler = mPendingIdleHandlers[i];
45                 mPendingIdleHandlers[i] = null; // release the reference to the handler
46                 
47                 boolean keep = false;
48                 try {
49                     keep = idler.queueIdle();
50                 } catch (Throwable t) {
51                     Log.wtf(TAG, "IdleHandler threw exception", t);   
52                 }
53                 
54                 if (!keep) {
55                     synchronozed(this) {
56                         mIdleHandlers.remove(idler);
57                     }
58                 }
59             }
60             
61             // Reset the idle handler count to 0 so we do not run them again.
62             pendingIdleHandlerCount = 0;
63             
64             // While calling an idle handler, a new message could have ben delivered
65             // so go back and look again for a pending message without waiting.
66             nextPollTimeoutMills = 0;
67         }
68     }
69 }
```

&#8195;&#8195;当前线程执行到第16行的if语句时，就说明当前线程准备要进入睡眠等待状态了。

&#8195;&#8195;第26行的if语句首先检查变量pendingIdleHandlerCount的值是否小于0并且mMessages为空或者mMessages的时间大于当前时间。如果为true，那么第28行代码就会获得所有注册到当前线程的消息队列中的空闲消息处理器的个数，并且保存在变量pendingIdleHandlerCount中。接着第30行的if语句再检查变量pendingIdleHandlerCount的值是否小于等于0。只有在两种情况下，变量pendingIdleHandlerCount的值才会等于0，其总，第一种情况是没有空闲消息处理器注册到当前线程的消息队列中；第二种情况是之前已经发送过一个线程空闲消息了。如果变量pendingIdleHandlerCount的值等于0，那么当前线程就不会再发出一个线程空闲消息了。

&#8195;&#8195;从这里就可以看出，在MessageQueue类的成员函数next的一次调用中，一个线程最多只会发出一个线程空闲消息。

```doc
这并不意味着一个线程最多只会发出一个线程空闲消息，因为MessageQueue类的成员函数next在一个线程的消息循环过程中，会不断地被调用，而每一次调用都有可能发出一个线程空闲消息。
```

&#8195;&#8195;第35行到第38行代码首先将注册到当前线程的消息队列中的空闲消息处理器拷贝到一个空闲消息处理器数组mPendingIdleHandlers中，接着第43行到第59行的for循环再依次调用这个数组中的每一个空闲消息处理器的成员函数queueIdle来接收一个线程空闲消息。

&#8195;&#8195;最后，第62行代码将变量pendingIdleHandlerCount的值重新设置为0，这样就可以保证在MessageQueue类的成员函数next的一次调用中，一个线程最多只会发出一个线程空闲消息。

至此，一个线程空闲消息的处理过程就分析完成了。通过注册空闲消息处理器，我们就可以把一些不重要的或者不紧急的事情放在线程空闲的时候来执行，这样就可以充分地利用消息的空闲时间。








