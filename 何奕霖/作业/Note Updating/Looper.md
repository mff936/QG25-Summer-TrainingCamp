# Kotlin Looper 详细教程

## 目录
1. [什么是Looper](#什么是looper)
2. [Looper的核心概念](#looper的核心概念)
3. [Handler与Looper的关系](#handler与looper的关系)
4. [消息队列机制](#消息队列机制)
5. [实际应用示例](#实际应用示例)
6. [高级用法](#高级用法)
7. [最佳实践](#最佳实践)
8. [常见问题](#常见问题)

## 什么是Looper

Looper是Android中用于实现线程间通信的核心机制之一。它允许一个线程处理来自其他线程的消息，实现异步编程模式。

### Looper的作用
- **消息循环**：Looper会不断从消息队列中取出消息并处理
- **线程通信**：允许不同线程之间通过消息进行通信
- **异步处理**：避免阻塞主线程，提高应用响应性

## Looper的核心概念

### 1. Looper
Looper是消息循环的核心，每个线程只能有一个Looper。

```kotlin
// 主线程的Looper（Android系统自动创建）
val mainLooper = Looper.getMainLooper()

// 当前线程的Looper
val currentLooper = Looper.myLooper()
```

### 2. MessageQueue
消息队列，存储待处理的消息。

```kotlin
// 获取Looper的消息队列
val messageQueue = looper.queue
```

### 3. Handler
消息处理器，负责发送和处理消息。

```kotlin
// 创建Handler
val handler = Handler(Looper.getMainLooper())
```

## Handler与Looper的关系

Handler和Looper是紧密相关的：

```kotlin
// 1. Handler需要绑定到Looper
val handler = Handler(looper)

// 2. Handler发送消息到Looper的消息队列
handler.sendMessage(message)

// 3. Looper从队列取出消息，调用Handler的handleMessage方法
override fun handleMessage(msg: Message) {
    // 处理消息
}
```

## 消息队列机制

### 消息的发送和处理流程

```kotlin
// 1. 创建消息
val message = Message.obtain()
message.what = MSG_TYPE
message.obj = data

// 2. 发送消息
handler.sendMessage(message)

// 3. Looper从队列取出消息
// 4. 调用Handler的handleMessage方法
override fun handleMessage(msg: Message) {
    when (msg.what) {
        MSG_TYPE -> {
            val data = msg.obj as? String
            // 处理消息
        }
    }
}
```

### 消息类型

```kotlin
// 普通消息
handler.sendMessage(message)

// 延迟消息
handler.sendMessageDelayed(message, 1000L)

// 定时消息
handler.sendMessageAtTime(message, SystemClock.uptimeMillis() + 1000L)

// Runnable任务
handler.post { 
    // 执行任务
}

// 延迟Runnable任务
handler.postDelayed({
    // 执行任务
}, 1000L)
```

## 实际应用示例

### 示例1：基本Looper线程

```kotlin
class BasicLooperThread : Thread() {
    private var looper: Looper? = null
    private var handler: Handler? = null
    
    override fun run() {
        // 1. 准备Looper
        Looper.prepare()
        
        // 2. 获取Looper
        looper = Looper.myLooper()
        
        // 3. 创建Handler
        handler = object : Handler(looper!!) {
            override fun handleMessage(msg: Message) {
                // 处理消息
                when (msg.what) {
                    MSG_TYPE -> {
                        val data = msg.obj as? String
                        Log.d("Looper", "处理消息: $data")
                    }
                }
            }
        }
        
        // 4. 开始消息循环
        Looper.loop()
    }
    
    fun sendMessage(data: String) {
        handler?.let { h ->
            val msg = h.obtainMessage(MSG_TYPE, data)
            h.sendMessage(msg)
        }
    }
    
    fun quit() {
        looper?.quit()
    }
}
```

### 示例2：带回调的Handler

```kotlin
class CallbackHandler : Handler.Callback {
    override fun handleMessage(msg: Message): Boolean {
        when (msg.what) {
            MSG_TYPE -> {
                val data = msg.obj as? String
                Log.d("Callback", "处理消息: $data")
                return true // 表示已处理
            }
        }
        return false // 表示未处理
    }
}

// 使用回调Handler
val handler = Handler(Looper.getMainLooper(), CallbackHandler())
```

## 高级用法

### 1. 消息优先级

```kotlin
// 高优先级消息（立即处理）
handler.sendMessageAtFrontOfQueue(highPriorityMessage)

// 普通消息
handler.sendMessage(normalMessage)

// 低优先级消息（延迟处理）
handler.sendMessageDelayed(lowPriorityMessage, 1000L)
```

### 2. 消息移除

```kotlin
// 移除所有消息
handler.removeCallbacksAndMessages(null)

// 移除特定类型的消息
handler.removeMessages(MSG_TYPE)

// 移除特定Runnable
handler.removeCallbacks(runnable)
```

### 3. 同步屏障

```kotlin
// 插入同步屏障
val token = handler.postSyncBarrier()

// 移除同步屏障
handler.removeSyncBarrier(token)
```

### 4. 异步消息

```kotlin
// 创建异步Handler
val asyncHandler = Handler(Looper.getMainLooper(), null, true)

// 发送异步消息
val asyncMessage = Message.obtain()
asyncMessage.setAsynchronous(true)
asyncHandler.sendMessage(asyncMessage)
```

## 最佳实践

### 1. 内存泄漏防护

```kotlin
class SafeHandler(looper: Looper) : Handler(looper) {
    private val weakReference = WeakReference<Activity>(activity)
    
    override fun handleMessage(msg: Message) {
        val activity = weakReference.get()
        if (activity != null && !activity.isFinishing) {
            // 处理消息
        }
    }
}
```

### 2. 线程安全

```kotlin
class ThreadSafeLooper {
    private val lock = Object()
    private var handler: Handler? = null
    
    fun getHandler(): Handler? {
        synchronized(lock) {
            return handler
        }
    }
    
    fun setHandler(newHandler: Handler) {
        synchronized(lock) {
            handler = newHandler
        }
    }
}
```

### 3. 优雅退出

```kotlin
class GracefulLooperThread : Thread() {
    private var looper: Looper? = null
    private var handler: Handler? = null
    private val isRunning = AtomicBoolean(false)
    
    override fun run() {
        Looper.prepare()
        looper = Looper.myLooper()
        handler = Handler(looper!!)
        isRunning.set(true)
        Looper.loop()
    }
    
    fun quit() {
        if (isRunning.get()) {
            looper?.quit()
            isRunning.set(false)
        }
    }
}
```

## 常见问题

### 1. 为什么主线程不需要手动调用Looper.prepare()？

Android系统在启动主线程时已经自动调用了`Looper.prepare()`，所以主线程可以直接创建Handler。

### 2. 如何避免Handler内存泄漏？

- 使用静态内部类
- 使用WeakReference
- 在Activity销毁时移除所有消息

### 3. Looper.loop()会阻塞线程吗？

是的，`Looper.loop()`会阻塞当前线程，直到调用`Looper.quit()`。

### 4. 如何实现消息优先级？

可以通过`sendMessageAtFrontOfQueue()`将高优先级消息插入队列头部。

### 5. Handler和Thread的关系？

Handler必须绑定到有Looper的线程，一个线程可以有多个Handler，但一个Handler只能绑定到一个Looper。

## 总结

Looper是Android异步编程的核心机制，通过消息队列实现线程间通信。掌握Looper的使用对于开发高性能的Android应用至关重要。

关键要点：
1. Looper负责消息循环
2. Handler负责消息发送和处理
3. MessageQueue存储消息
4. 主线程自动有Looper
5. 工作线程需要手动准备Looper
6. 注意内存泄漏和线程安全

通过本教程的学习，您应该能够熟练使用Kotlin Looper进行异步编程了.