# 在协程中使用Flow

## 协程的相关概念

> Essentially, coroutines are light-weight threads.

协程 Coroutines 可以理解成**轻量级的线程**，其本质是对于线程池 API 的封装，是一种多任务并发的框架

协程的优点：
- 用同步方式写异步代码
- 自动切换线程，降低犯错几率
- 替代回调，解决回调嵌套的问题
- 使用 Jetpack 架构组件中的协程框架（LifeCycle/ViewModel）追踪，避免泄露

以下是个多层嵌套的网络请求的例子。可以看出使用协程写的话，没有回调，而且代码结构相当清晰

```kotlin
coroutineScope.launch(Dispatchers.Main) {       
    // 网络请求：获取 token
    val token = api.getToken()     
    // 网络请求：获取用户信息，需要返回的 token 作为上行参数
    val user = api.getUser(token)   
    // 回到主线程通过获取到的用户信息更新UI            
    updateUI(user)             
}
```

协程本身虽然是轻量级的，但是由于本质其实是运行在线程池中，性能与直接使用线程池并无太大差别

### 协程作用域 CoroutineScope

为了避免协程泄露，Kotlin 引入了结构化并发 structured concurrency。在 Kotlin 中，**协程必须运行在 CoroutineScope 中**。为了保证所有的协程都被追踪到，Kotlin **不允许你在没有 CoroutineScope 的情况下开启新的协程**（你可以把 CoroutineScope 想象成被封装过的轻量级的 ExecutorService）。CoroutineScope 会追踪所有的协程，并且它也可以取消所有由他开启的协程。

```kotlin
// 开启协程
val job = GlobalScope.launch {    
    delay(5000)
}
```

### 调度器

- Dispatchers.Main：使用这个调度器在 Android 主线程上运行一个协程。可以用来更新UI 。在UI线程中执行
- Dispatchers.IO：这个调度器被优化在主线程之外执行磁盘或网络 I/O。在线程池中执行
- Dispatchers.Default：这个调度器经过优化，可以在主线程之外执行 cpu 密集型的工作。例如对列表进行排序和解析 JSON。在线程池中执行。
- Dispatchers.Unconfined：在调用的线程直接执行。

```kotlin
// 从 IO 线程开启协程
coroutineScope.launch(Dispatchers.IO) {
    ...
}

// 从主线程开启协程
coroutineScope.launch(Dispatchers.Main) {
    ...
}
```

### 挂起suspend 

挂起：指的是当前的这个协程从正在执行的线程上脱离，当前线程不再管这个协程接下去要做什么事情了。然后协程会在 **suspend 指定的线程上执行代码**。而在suspend 函数执行完后，协程会自动把线程再切换回来

`suspend`这个关键字加到函数体的最前面，**表示该方法是可挂起的**。被suspend 修饰的函数叫挂起函数

这一步是协程底层框架实现的，这个切换回来的操作叫 resume（恢复），只在协程内部有效。使用时只需要在函数定义上添加`suspend`关键字即可。第一个例子中的`delay`函数就是一个挂起函数

### withContext

这也是一个系统定义的挂起函数，指定调度器后可以切换到指定线程，并在其闭包内执行结束后自动切回去执行。相对于使用`launch`进行调度，减少了层级嵌套
```kotlin
// 第一种写法
coroutineScope.launch(Dispatchers.IO) {
    ...
    launch(Dispatchers.Main){
        ...
        launch(Dispatchers.IO) {
            ...
            launch(Dispatchers.Main) {
                ...
            }
        }
    }
}

// 通过第二种写法来实现相同的逻辑
coroutineScope.launch(Dispatchers.Main) {
    ...
    withContext(Dispatchers.IO) {
        ...
    }
    ...
    withContext(Dispatchers.IO) {
        ...
    }
    ...
}
```
`withContext` 这个函数本身就是一个协程框架内部定义好的挂起函数，其接收一个 `Dispatcher` 参数，由这个参数指定要切换的线程。我们想要自己实现一个挂起函数，只加 `suspend` 关键字是不行的，需要直接或者间接调用协程框架实现的挂起函数（比如`withContext`）才行

```kotlin
// 直接修饰没有任何作用
suspend fun getUser(id: String) {
    ...
}

// 必须调用协程框架实现的挂起函数
suspend fun getUser(id: String) = withContext(Dispatchers.IO) {
    ...
}
```

### 协程结合 Android Jetpack 使用

协程结合LifeCycle：引用 `androidx.lifecycle:lifecycle-runtime-ktx`
```kotlin
lifecycleScope.launch { 
    val deferred = async(Dispatchers.Default) {
    		// 网络请求
  	}
  	val result = deferred.await()
}
```

协程结合LiveData：引用 `androidx.lifecycle:lifecycle-livedata-ktx`

```kotlin
val data: LiveData<Repos> = liveData { 
    try { 
        val repos = api.getRepos() 
        // 通过emit()更新livedata 
        emit(repos) 
    } catch (e: Exception) { 
        ... 
    } 
}
```

协程结合ViewModel：引用 `androidx.lifecycle:lifecycle-viewmodel-ktx`
```kotlin
viewModelScope.launch {    
    try {        
        val netBean = service.userLogin(userName, userPassword)        
        if (netBean.errorCode == 0 && netBean.data != null) {           
            // 更新liveData            
            userLoginData.value = netBean.data        
        } else {            
            // 登录失败         
        }    
    } catch (e: Exception) {       
        // 登录失败        
        userLoginData.value = null   
    }
}
```

## Flow

> Flow - cold asynchronous stream with flow builder and comprehensive operator set (filter, map, etc)

- 类似于 RxJava，可以理解成为kotlin版本的RxJava（其本身设计参考了 RxJava）
- 是一种**有序**的冷流（冷流：调用了`subscribe() / collected()`后才会执行代码）
- 协作取消：和协程一致，只有在可取消的挂起函数中才可以被取消

对于`Flow`接口，只定义了一个`collect`函数

```kotlin
public interface Flow<out T> {
    @InternalCoroutinesApi
    public suspend fun collect(collector: FlowCollector<T>)
}
```

对应来说，可以理解为`collect()`相当于 RxJava 中的 `subscribe()`，`emit()`相当于 RxJava 中的 `emitter.onNext()`

Flow与协程的关系：**必须在协程中使用**

### 创建

```kotlin
// 1. flow builder
flow{
    ...
    emit(x)
}

// 2. 使用 flowOf
flowOf(1,2,3,4,5)

// 3. 集合通过 asFlow 转换
listOf(1,2,3,4,5).asFlow()
```


### 线程切换

Flow使用`flowOn`来实现线程的切换，实际上改变的是协程本身对应的线程

| 操作 |FLow  |RxJava  |
| --- | --- | --- |
|改变发射（订阅）线程 |flowOn  |subscribeOn  |
| 改变消费线程 | -- | observeOn |

之所以没有改变消费线程的操作，是因为**在启动协程时指定的调度器就确认好了消费线程**

```kotlin
GlobalScope.launch(Dispathers.Main) {
    flow {
        (1,2,3).foreach {
            emit(it)
            delay(1000)
        }
    }
    .flowOn(Dispatchers.IO)
    .collect { 
        ...
    }
}
```

### 异常处理

Flow中的异常可以通过`catch`操作符来完成，其本质是对声明式捕获`try...catch...`封装的一个扩展方法

```kotlin
GlobalScope.launch(Dispathers.Main) {
    flow {
        ...
    }
    .catch{ e ->
        ...
    }
    .collect { 
        ...
    }
}
```

### 背压

与RxJava一样，Flow 中也支持类似的背压策略
- `buffer` 对应 Flowable 的 
`BackpressureStrategy.BUFFER` 策略（即 Observable 的默认表现），会消费所有事件，不会抛出异常，但是可能导致OOM
- `conflate` 对应 Flowable 的 
`BackpressureStrategy.LATEST ` 策略，缓存池满了后新数据会覆盖老数据，可用于跳过中间值
- `collectLatest` 只处理最新数据，如果前一个正在处理中但是还没处理完的时候后一个过来了，前一个数据将会被**直接取消处理**

### 末端操作符

之前例子中都是用 `collect` 消费 Flow 的数据。`collect` 是最基本的末端操作符，功能与 RxJava 的 `subscribe` 类似。

而 Flow 除了 `collect` 之外，还有其他常见的末端操作符
- 集合类型转换操作，包括 `toList`、`toSet` 等
- 聚合操作，包括将 Flow 规约到单值的 `reduce`、`fold` 等操作，以及获得单个元素的操作包括 `single`、`singleOrNull`、`first` 等
- 上面这些末端操作符最终还是调用的`collect`

> 实际上，识别是否为末端操作符，还有一个简单方法，**由于 Flow 的消费端一定需要运行在协程当中，因此末端操作符都是挂起函数**

### 与 RxJava 操作符对比


|  |Flow  |RxJava  |
| --- | --- | --- |
| 完成 |onCompletion  |onComplete  |
| 转换 |map  |map  |
|  |take  |take  |
|  |filter  |filter  |
|组合  |zip  |zip  |
|  |combine  |combine  |
| | merge|merge|
| 展平 |flatMapMerge  |flatMap  |
|  |flatMapConcat  |concatMap  |
|  |flatMapLatest  |  |

由此可见 Flow 中基本涵盖了 RxJava 中常见的操作符，容易上手

举一个 `zip` 操作符的例子：

```kotlin
fun onOperator(view: View) {
		GlobalScope.launch {
    		mockNetWork1()
    				.zip(mockNetWork2()) { a, b -> a + b }
    				.collect {
      					Log.e(tag, "a+b = $it")
    				}
  	}
}

private suspend fun mockNetWork1() = flow {
  	delay(2000)
  	emit(1)
}

private suspend fun mockNetWork2() = flow {
  	delay(1000)
  	emit(1)
}
```

*不过个人更喜欢这样写合并的网络请求*

```kotlin
coroutineScope.launch(Dispatchers.Main) {
    // 使用 async 函数获取返回值 await
    val avatar = async { api.getAvatar(user) }
    val logo = async { api.getCompanyLogo(user) }
    // 合并结果 suspendingMerge 自定义挂起函数
    val result = suspendingMerge(avatar.await(), logo.await())  
    // 更新 UI      
    updateUI(result)
}
```

### 在MVVM架构中的使用

之前在 ViewModel 中使用协程操作时，一般这么写
```kotlin
val result: MutableLiveData<String> = MutableLiveData<String>()

fun run() {
    viewModelScope.launch {
        result.value = doComputation()
    }
}
```
在使用了 Flow 之后，我们可以在 LiveData 上做文章，写成下面这种形式

```kotlin
val result: LiveData<String> = flow {
    emit(doComputation())
}.asLiveData()
```

或者干脆更简单一些

```kotlin
val result: LiveData<String> = liveData {
    emit(doComputation())
}
```

查看[官方文档](http://www.kotlincn.net/docs/reference/coroutines/flow.html)发现更多 Flow 的使用方法

