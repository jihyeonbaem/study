코루틴 빌더 함수가 호출되면 새로운 코루틴을 생성하고, 코루틴을 추상화한 Job 객체를 반환한다.

Job 객체는 코루틴의 상태를 추적하고 제어할 수 있다.

코루틴은 일시 중단 가능한 작업

일시 중단과 재개

## join을 사용한 코루틴 순차처리

순차 처리가 필요한 경우 → ex) 데이터베이스 작업 순서, 캐싱된 토큰 값 업데이트 이후 요청

`Job.join()` 함수 : (Q) 코루틴의 실행이 완료될 때 까지 호출된 Job을 가진 코루틴을 일시중단

```kotlin
public suspend fun join()
```

- join 함수를 호출한 코루틴만 일시 중단한다. [아래 코드에서는 runBlocking의 코루틴]

  join의 대상인 updateTokenJob 코루틴이 완료될 때까지

- 이미 실행 중인 다른 코루틴들을 일시 중단하지 않는다. [updateTokenJob, independentJob은 정상실행]

~~실행을 여러번 반복하면 “독립 작업 실행”이 “토큰 업데이트 시작”보다 먼저 출력되는 경우를 볼 수 있다.~~

~~→ 코루틴의 실행순서는 보장되지 않는다. CoroutineDispatcher의 내부 스케줄러가 결정한다.~~

```kotlin
fun main() = runBlocking<Unit> {
    val updateTokenJob: Job = launch(Dispatchers.IO) {
        printlnCurrentThreadNameWithText("토큰 업데이트 시작")
        delay(1000L)
        printlnCurrentThreadNameWithText("토큰 업데이트 완료")
    }

    val independentJob: Job = launch(Dispatchers.IO) {
        printlnCurrentThreadNameWithText("독립 작업 실행")
    }

    updateTokenJob.join()

    val networkJob: Job = launch(Dispatchers.IO) {
        printlnCurrentThreadNameWithText("네트워크 요청")
    }
}

// output
[DefaultDispatcher-worker-2 @coroutine#3] 독립 작업 실행
[DefaultDispatcher-worker-1 @coroutine#2] 토큰 업데이트 시작
<1초 뒤>
[DefaultDispatcher-worker-1 @coroutine#2] 토큰 업데이트 완료
<updateTokenJob 이 완료된(isCompleted) 뒤>
[DefaultDispatcher-worker-1 @coroutine#4] 네트워크 요청
```

### joinAll

join함수를 여러개 사용하는 것과 동일하다.

- 서로 독립적인 코루틴을 여러개 병렬로 실행 한 후 각 코루틴이 모두 끝날 때까지 기다렸다가 작업해야할 때 사용

```kotlin
public suspend fun joinAll(vararg jobs: Job): Unit = jobs.forEach { it.join() }
```

예시 ) A1~10 까지 작업을 joinAll로 완료한 뒤, B1~10 까지 launch 실행 할 수가 있음

## CoroutineStart.LAZY

코루틴을 원하는 시점에 실행하기

- `launch` 함수를 사용하면 사용가능한 스레드가 있는 경우 코루틴이 즉시 실행된다.
- 코루틴을 생성할 때 나중에 시작하기 위해서 launch의 start 인자에 `CoroutineStart.LAZY`를 넘겨준다.
    - `Job.start()`를 명시적으로 호출해야만 실행 지연된 코루틴이 시작된다.

CoroutineStart는 enum class로 4가지 유형이 있다.

`CoroutineStart.DEFAULT`

```kotlin
fun main() = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()

    val immediateJob: Job = launch(
        start = CoroutineStart.DEFAULT, // 기본 값
    ) {
        println("[지난 시간 ${System.currentTimeMillis() - startTime}ms]")
    }
}

// output
[지난 시간 2ms]
```

`CoroutineStart.LAZY`

```kotlin
fun main() = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()

    val lazyJob: Job = launch(
        start = CoroutineStart.LAZY,
    ) {
        println("[지난 시간 ${System.currentTimeMillis() - startTime}ms]")
    }

    delay(1000L)
    lazyJob.start()
}

// output
[지난 시간 1010ms]
```

`CoroutineStart.UNDISPATCHED`

- 첫번째 일시중단 시점까지 코루틴을 즉시 현재 스레드에서 실행됨
- 일시 중단 이후 재개할 때, 적용된 디스패처를 사용함

    ```kotlin
    fun main() = runBlocking<Unit> {
    
        printlnCurrentThreadNameWithText("실행1")
        val job: Job = launch(
            context = Dispatchers.IO,
            start = CoroutineStart.UNDISPATCHED,
        ) {
            printlnCurrentThreadNameWithText("실행2")
            delay(1000L)
            printlnCurrentThreadNameWithText("실행3")
        }
    }
    
    // output
    [main @coroutine#1] 실행1
    [main @coroutine#2] 실행2
    [DefaultDispatcher-worker-1 @coroutine#2] 실행3
    ```


`CoroutineStart.ATOMIC`  @DelicateCoroutinesApi

- ~~코루틴의 body가 실행 시작하기 전까지는~~
    - ~~취소할 수 없는 방식으로 실행~~
    - ~~코루틴이 취소되더라도 실행이 시작되도록 보장~~
- ~~코루틴 body가 시작된 이후에는 cancellation이 일반적으로 동작함~~

```kotlin
    @InternalCoroutinesApi // 내부 API
    public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
        when (this) {
            DEFAULT -> block.startCoroutineCancellable(receiver, completion)
            ATOMIC -> block.startCoroutine(receiver, completion)
            UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
            LAZY -> Unit // will start lazily
        }

```

### launch 함수 1뎁스 내부구현

1. 새로운 CoroutineContext 생성
2. 코루틴 인스턴스 생성

   lazy → LazyStandaloneCoroutine(newContext, block)

   else → StandaloneCoroutine(newContext, active = true)

3. 코루틴 시작

   coroutine.start(start, coroutine, block)

4. Job 반환

   return coroutine


```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

- ~~2,3 뎁스~~

    ```kotlin
    private open class StandaloneCoroutine(
        parentContext: CoroutineContext,
        active: Boolean
    ) : AbstractCoroutine<Unit>(parentContext, initParentJob = true, active = active) {
        override fun handleJobException(exception: Throwable): Boolean {
            handleCoroutineException(context, exception)
            return true
        }
    }
    
    private class LazyStandaloneCoroutine(
        parentContext: CoroutineContext,
        block: suspend CoroutineScope.() -> Unit
    ) : StandaloneCoroutine(parentContext, active = false) {
        private val continuation = block.createCoroutineUnintercepted(this, this)
    
        override fun onStart() {
            continuation.startCoroutineCancellable(this)
        }
    }
    ```

    ```kotlin
    @OptIn(InternalForInheritanceCoroutinesApi::class)
    @InternalCoroutinesApi
    public abstract class AbstractCoroutine<in T>(
        parentContext: CoroutineContext,
        initParentJob: Boolean,
        active: Boolean
    ) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
    
        init {
            if (initParentJob) initParentJob(parentContext[Job])
        }
    
        @Suppress("LeakingThis")
        public final override val context: CoroutineContext = parentContext + this
    
        public override val coroutineContext: CoroutineContext get() = context
    
        override val isActive: Boolean get() = super.isActive
    
        protected open fun onCompleted(value: T) {}
    
        protected open fun onCancelled(cause: Throwable, handled: Boolean) {}
    
        override fun cancellationExceptionMessage(): String = "$classSimpleName was cancelled"
    
        @Suppress("UNCHECKED_CAST")
        protected final override fun onCompletionInternal(state: Any?) {
            if (state is CompletedExceptionally)
                onCancelled(state.cause, state.handled)
            else
                onCompleted(state as T)
        }
    
        public final override fun resumeWith(result: Result<T>) {
            val state = makeCompletingOnce(result.toState())
            if (state === COMPLETING_WAITING_CHILDREN) return
            afterResume(state)
        }
    
        protected open fun afterResume(state: Any?): Unit = afterCompletion(state)
    
        internal final override fun handleOnCompletionException(exception: Throwable) {
            handleCoroutineException(context, exception)
        }
    
        internal override fun nameString(): String {
            val coroutineName = context.coroutineName ?: return super.nameString()
            return "\"$coroutineName\":${super.nameString()}"
        }
    
        public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
            start(block, receiver, this)
        }
    }
    
    ```


## cancel

코루틴을 실행할 필요가 없어지면 즉시 취소해야한다.

불필요한 코루틴이 지속적으로 스레드를 사용하면 앱 성능이 저하된다.

Job 객체는 코루틴을 취소할 수 있는 cancel 함수를 제공한다.

### longJob [no cancel]

```kotlin
fun main() = runBlocking<Unit> {
    printlnCurrentThreadNameWithText("실행")

    val longJob: Job = launch(
        context = Dispatchers.Default,
        start = CoroutineStart.DEFAULT,
    ) {
        repeat(100) {
            delay(1000L) // 1초 지연
            printlnCurrentThreadNameWithText("실행 ${it + 1}")
        }
        printlnCurrentThreadNameWithText("실행 끝")
    }
}

// output
[main @coroutine#1] 실행
<1초 뒤>
[DefaultDispatcher-worker-1 @coroutine#2] 실행 1
<1초 뒤>
[DefaultDispatcher-worker-1 @coroutine#2] 실행 2
...
<1초 뒤>
[DefaultDispatcher-worker-1 @coroutine#2] 실행 100
[DefaultDispatcher-worker-1 @coroutine#2] 실행 끝
```

### longJob [cancel]

```kotlin
fun main() = runBlocking<Unit> {
    printlnCurrentThreadNameWithText("실행")

    val longJob: Job = launch(
        context = Dispatchers.Default,
        start = CoroutineStart.DEFAULT,
    ) {
        repeat(100) {
            delay(1000L) // 1초 지연
            printlnCurrentThreadNameWithText("실행 ${it + 1}")
        }
        printlnCurrentThreadNameWithText("실행 끝")
    }

    delay(3500L)
    longJob.cancel()
}
// output
[main @coroutine#1] 실행
<1초 뒤>
[DefaultDispatcher-worker-1 @coroutine#2] 실행 1
<1초 뒤>
[DefaultDispatcher-worker-1 @coroutine#2] 실행 2
<1초 뒤>
[DefaultDispatcher-worker-1 @coroutine#2] 실행 3
<0.5초 뒤 longJob.cancel() 호출 후 프로세스 종료됨>
Process finished with exit code 0
```

### cancel 함수 1뎁스 내부구현

```kotlin
// 지정된 Job(모든 하위 작업을 포함한)을 취소한다.
// 디버깅을 위해 취소 사유를 인자로 전달할 수 있다.
public abstract fun cancel(cause: java.util.concurrent.CancellationException? /* from: kotlinx.coroutines.CancellationException? */ = COMPILED_CODE): kotlin.Unit

public fun Job.cancel(message: String, cause: Throwable? = null): Unit = 
		cancel(CancellationException(message, cause))

public fun cancel(cause: CancellationException? = null)
```

### cancelAndJob을 사용한 순차 처리

- Job.cancel() 함수는 해당 Job을 가진 코루틴에 취소를 요청한다.
- 코루틴 취소는 미래 어느 시점에 취소된다.
    - 특정 코루틴이 취소된 이후에 실행되어야하는 함수가 있을 경우 cancel 함수만으로 순차성을 보장할 수 없다.
- cancelAndJob을 사용해 코루틴이 취소되고 완료되는 시점까지 일시중단되도록 보장한다.

  cancel 과 Job 이 결합된 형태

    ```kotlin
    public suspend fun Job.cancelAndJoin() {
        cancel()
        return join()
    }
    ```


### cancel 확인

cancel 함수를 사용하는 것은 취소를 요청하는 것이고, 실제 코루틴이 즉시 취소 되는 것은 아니다.

→ 코루틴이 취소 요청을 확인하는 시점에 취소된다. [확인할 수 있는 시점이 없다면, 취소될 수 없다.]

일반적으로, 코루틴은 “일시 중단”, “실행 대기” 시점에 취소를 확인한다.

- 코루틴이 지속적인 작업을 실행 → 취소 요청을 확인할 수 없음

    ```kotlin
    fun main() = runBlocking<Unit> {
        val job: Job = launch(context = Dispatchers.Default) {
            while (true) {
                printlnCurrentThreadNameWithText("작업중")
            }
            printlnCurrentThreadNameWithText("작업 끝")
        }
        
        job.cancel()
    }
    
    // output
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    ...
    ```

- delay

  while문이 반복될 때마다 delay(1)로 1ms 동안 일시중단되어 취소를 확인할 수 있다.

    ```kotlin
    fun main() = runBlocking<Unit> {
        val job: Job = launch(context = Dispatchers.Default) {
            while (true) {
                printlnCurrentThreadNameWithText("작업중")
                delay(1)
            }
            printlnCurrentThreadNameWithText("작업 끝")
        }
    		
    		delay(500L)
        job.cancel()
    }
    // output
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    ...
    <0.5초 뒤 cancel()로 코루틴이 완료된 후 프로세스 종료>
    Process finished with exit code 0
    ```

    - delay 내부구현
        - suspendCancellableCoroutine : suspendCoroutine처럼 코루틴을 일시중단하고, CancellableContinuation를 제공한다.
        - scheduleResumeAfterDelay : 딜레이가 끝난 후 재개됨

          *schedule: 나중에 실행(재개)되도록 예약하는 것


        ```kotlin
        public suspend fun delay(timeMillis: Long) {
            if (timeMillis <= 0) return // don't delay
            return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->
                // if timeMillis == Long.MAX_VALUE 이면 awaitCancellation처럼 영원히 기다린다.
                if (timeMillis < Long.MAX_VALUE) {
                    cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)
                }
            }
        }
        ```
        
        - `awaitCancellation()` 취소될 때까지 delay
        
        ```kotlin
        public suspend fun awaitCancellation(): Nothing = suspendCancellableCoroutine {}
        ```


- yield (=양보하다) [석준님도 사용해본적이 없다고함]

  코루틴을 일시 중단하고 즉시 추가 실행을 위해 예약한다.

  다른 코루틴이 실행될 수 있는 기회를 주어야할 때 사용

    ```kotlin
    fun main() = runBlocking<Unit> {
        val job: Job = launch(context = Dispatchers.Default) {
            while (true) {
                printlnCurrentThreadNameWithText("작업중")
                yield()
            }
            printlnCurrentThreadNameWithText("작업 끝")
        }
        delay(500L)
        job.cancel()
    }
    // output
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    ...
    <0.5초 뒤 cancel()로 코루틴이 완료된 후 프로세스 종료>
    Process finished with exit code 0
    ```

    - yield 내부구현

        ```kotlin
        public suspend fun yield(): Unit = suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
            val context = uCont.context
            context.ensureActive() // 취소
            val cont = uCont.intercepted() as? DispatchedContinuation<Unit> ?: return@sc Unit
            if (cont.dispatcher.safeIsDispatchNeeded(context)) {
                cont.dispatchYield(context, Unit)
            } else {
                val yieldContext = YieldContext()
                cont.dispatchYield(context + yieldContext, Unit)
                if (yieldContext.dispatcherWasUnconfined) {
        
                    return@sc if (cont.yieldUndispatched()) COROUTINE_SUSPENDED else Unit
                }
            }
            COROUTINE_SUSPENDED
        }
        ```


delay, yield를 사용하면 불필요하게 스레드를 일시중단하게 되는 문제점이 발생 → 비효율

- CoroutineScope.isActive

  CoroutineScope는 코루틴이 활성화 됐는지 확인할 수 있는 Boolean 타입 isActive를 제공

  코루틴 “취소 요청”시 isActive 값이 false로 변함

  → 불필요한 스레드 중단 없이 cancel호출시 작업을 취소할 수 있다.

    ```kotlin
    fun main() = runBlocking<Unit> {
        printlnCurrentThreadNameWithText("실행")
    
        val job: Job = launch(context = Dispatchers.Default) {
            while (this.isActive) {
                printlnCurrentThreadNameWithText("작업중")
            }
            printlnCurrentThreadNameWithText("작업 끝")
        }
        delay(1000L)
        job.cancel()
    }
    // output
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    ...
    [DefaultDispatcher-worker-1 @coroutine#2] 작업중
    <1초 뒤>
    [DefaultDispatcher-worker-1 @coroutine#2] 작업 끝
    ```


## CoroutineScope.isActive, isCancelled, isCompleted

코루틴의 상태 6가지

생성(New), 실행 중(Active), 실행 완료(Completed), 취소 중(Cancelling), 취소 완료(Cancelled) [135p 그림4-5참고]

- `생성` : 코루틴 빌더를 통해 코루틴을 생성, CoroutineStart.LAZY가 아닌 경우는 자동으로 실행중 상태로 넘어감
- `실행 중` : 실제 실행중 or 일시 중단 상태
- `실행 완료` : 코루틴의 모든 코드가 실행 완료된 경우
- `취소 중` : job.cancel() 등을 통해 코루틴에 취소가 요청됐을 경우 [코루틴은 계속 실행 중]
- `취소 완료` : 코루틴의 취소가 확인된 경우 취소 완료 [코루틴 더이상 실행x]

- isActive : 코루틴이 활성화 중인지 여부 [생성후o, 실행 중o, 취소 중x, 취소완료x, 실행완료x]
- isCancelled : 코루틴이 취소 요청됐는지 여부 [취소중o, 취소 완료o]
- isCompleted : 코루틴이 실행 완료됐는지 여부 [실행완료o, 취소 완료o]