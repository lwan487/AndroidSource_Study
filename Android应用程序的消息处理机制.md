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

&#8195;&#8195;C++层中的Looper队友两个类型为int的成员变量**mWakeReadPipeFd**和**mWakeWritePipeFd**，分别用来描述一个管道的的读端文件描述符和写端文件描述符。当一个线程的消息队列没有消息需要处理时，它就会在这个管道的读端文件描述符上进行睡眠等待，知道其他线程通过这个管道的写端文件描述符来唤醒它为止。

&#8195;&#8195;当调用Java层的Looper类的静态成员函数prepareMainLooper或者prepare来为一个线程创建一个消息队列时，Java层的Looper类就会在这个线程中创建一个Looper对象和一个MessageQueue对象。在创建Java层的MessageQueue对象的过程中，又会调用自身成员变量nativeInit在C++ 层中创建一个NativeMessageQueue对象和一个Looper对象。在创建C++层的Looper对象时，又会创建一个管道，这个管道的读端文件描述符和写端描述符就保存在它的成员变量mWakeReadPipedFd和mWakeWritePipedFd中。

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







