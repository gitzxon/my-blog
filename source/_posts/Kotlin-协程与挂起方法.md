---
title: Kotlin 协程与挂起方法
date: 2023-01-25 16:35:52
tags:
---

# Kotlin 协程与挂起方法

## 协程可以做啥？

1. 处理耗时任务。耗时任务放在主线程中会导致 UI 卡顿或者 ANR。作用等同于启动一个线程池执行耗时任务
2. 把回调式 API 变成顺序式，增加可读性，降低理解成本

因为处理耗时任务的用法，跟 java 的线程池比较类似，也比较简单，本文不做过多概述。后面着重聊聊利用协程以及挂起方法，把回调式 API 封装成顺序式 API。



## 协程和挂起方法的关系

说到协程，不得不提挂起方法。

挂起方法，是用`suspend`关键字修饰的方法。他俩的关系是，<font color=red>挂起方法必须在协程中执行，但是协程中执行的不要求必须是挂起方法</font>。挂起方法执行的时候，看起来像是代码「阻塞」在了挂起方法上，这就是「挂起」。后面的代码必须等挂起方法执行完成后，才能执行。

上面说，代码像是「阻塞」，更多是表示「代码停在当前这条语句上」。实际上，他跟线程的「阻塞」不一样。线程阻塞，对应的底层 CPU 也在等待。但是协程挂起的时候，对应的线程依然在工作，底层的 CPU 也依然在运转。这也是为什么我们说，基于协程的调度任务，会比基于线程的调度任务，效率高。



## 挂起方法的原理

原理上，编译器在编译阶段，会为挂起方法，生成有限状态机，来处理协程。

###接口声明

与协程交互，需要通过`interface Continuation` 对象。`Continuation`接口，从功能上，更像是有了更多信息、更多上下文的 callback。

[查看文档](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/)，`Continuation`相关定义如下 ：

属性：

abstract val context: [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/index.html)

包含了协程上下文信息。

方法：

abstract fun resumeWith(**result**: [Result](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/index.html)<T>)

表示协程结束。result 代表返回的结果 ，可以是成功的，也可以是失败的。

拓展方法：

1. fun <T> [Continuation](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/index.html)<T>.resume(**value**: T)
2. fun <T> [Continuation](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/index.html)<T>.resumeWithException(
     **exception**: [Throwable](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-throwable/index.html))

两个拓展方法其实是增加了语法糖，让代码编写更加灵活高效。

###Kotlin Compiler 对挂起函数的处理

编译期间，编译器会改变挂起方法的签名，为他后面添加一个参数：`completion: Continuation<Any?>`，并修改返回类型为 void。

```kotlin
// 我们写的 Kotlin 代码
suspend fun loginUser (userId: String, password: String): User (
  val user = userRemoteDataSource.loginUser (userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDo
}
  
// Kotlin Compiler 生成的代码的等效 Kotlin 代码
fun loginUser (userId: String, password: String, completion: Continuation<Any?>) {
  val user = userRemoteDataSource. loginUser (userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  completion.resume(userDb)
}
```

这时我们可能有个疑问：函数签名中返回值类型变成了 void，那结果怎么返回呢？注意这句话：

```kotlin
completion.resume(userDb)
```

返回值还在，只不过是通过 Continuation 对象包装了一下。



### 有限状态机

回到我们刚才说的，Kotlin 编译器会生成一个有限状态机，在哪呢？

如下，编译器会识别里面的可以挂起的方法，然后生成状态。状态的数量 = 挂起方法个数 + 1。后面有个加一，类似于小学时候学的植树问题。

```Kotlin
fun loginUser (userId: String, password: String, completion: Continuation<Any?>) {
  // Label 0
  -> first execution
  val user
  = userRemoteDataSource. loginUser (userId, password)
  // Label 1
  -> resumes from userRemoteDataSource
  val userDb = userLocalDataSource.logUserIn(user)
  // Label 2
  -> resumes from userLocalDataSource
  completion.resume (userDb)
}
```

生成的状态机，核心是一个 label + 一个 when。根据执行进度，label 值会被更改，然后根据 label 值，选择不同的分支，执行对应的代码。

```kotlin
fun loginUser (userId: String, password: String, completion: Continuation<Any?>) {
	when (label) {
		0 -> { // Label 0 -> first execution
			userRemoteDataSource.loginUser(userId,password)
		}
		1 -> { // Label 1 -> resumes from userRemoteDataSource
			userLocalDataSource.logUserIn(user)
		}
		2 -> { // Label 2 -> resumes from userLocalDataSource
			completion. resume (userDb)
		}
		else -> throw IllegalStateException(...)
	}
}
```



不过执行的过程中，数据是如何传输的？

数据有两个来源

1. 函数的参数
2. 执行过程中产生的中间变量

```kotlin
fun loginUser (userId: String?, password: String?, completion: Continuation<Any?>) {
	class LoginUserStateMachine(completion: Continuation<Anv?>) : CoroutineImpl (completion) {
		var user: User? = null
		var userDb: UserDb? = null
		var result: Anv? = null
		var label: Int = 0
		override fun invokeSuspend(result: Any?) {
			this.result = result
			loginUser(null, null, this)
		}
	}
}
```

编译器会帮我们生成一个内部类，名字叫 XxxStateMachine，也就是状态机的真容。所有参数，都以「内部类的 成员变量的形式」来表示。随着挂起方法的运行，不断的对各个成员变量进行赋值。同时，内部类中还有一个`invokeSuspend`方法，用于执行状态机。其中，前面所有参数都是 null，最后是 this，用于协程退出。	

这里有两个关键点：

1. 参数用内部类的成员变量来表示
2. invokeSuspend 方法作为入口，loginUser 前面的参数都是 null

这里我们可能有疑问，如果参数都是 null，那参数不是相当于都没传吗？参数都没有，得到的运算结果能正确吗？
且看下面代码：

```kotlin
{
	val continuation = completion as? LoginUserStateMachine?: LoginUserStateMachine(completion)
	when (continuation.label) {
		0 -> {
			throwOnFailure(continuation.result)
			continuation.label = 1
			userRemoteDataSource.loginUser(userId, password, continuation)	
		}
		1 -> {
			throwOnFailure(continuation.result)
			continuation.user = continuation.result as User
			continuation. label = 2
			userLocalDataSource.logUserIn(continuation.user, continuation)
		}
	    2 -> {
			throwOnFailure(continuation.result)
			continuation.userDb = continuation.result as UserDb
			continuation.cont.resume(continuation.userDb)
		}
		else -> throw IllegalStateException(...)
}
 
```

挂起函数在挂起和恢复的时候，数据都保存在了  continuation 中了（如果没执行过，continuation 为 null，则 new 一个新的）。所以，表面上看，所有的参数都是 null，好像没传参。其实是换了一种形式，以成员变量的方式，通过 completion 对象传过去的。
```
val continuation = completion as? LoginUserStateMachine?: LoginUserStateMachine(completion)
```

## 总结
我们首先聊了聊协程的两个常见用途，然后引出挂起方法以及状态机，最后通过展示 Kotlin 编译器生成代码的等价 Kotlin 代码，揭示了挂起方法暂停和恢复底层实现原理。
