## 비슷하지만 다른 launch vs async

코루틴 빌더를 통해 생성되는 코루틴

- `launch` 는 결괏값이 없는 코루틴 객체 Job을 반환한다.
- `async` 는 결괏값을 가진 코루틴 객체 Deferred<T>를 반환한다.
    - Deferred 객체를 통해 코루틴으로부터 결괏값을 받을 수 있다.

- `launch`  살펴보기

    ```kotlin
    /*
    launch는 새로운 코루틴을 생성한다.
    현재 스레드를 차단하지 않고 코루틴을 실행하고, 그 코루틴을 관리할 수 있는 [Job] 객체 반환
    Job 객체가 취소되면[Job.cancel 호출 후 cancelled] 코루틴도 취소된다.
    
    CoroutineContext는 [CoroutineScope]로부터 상속받는다.
    [context] 인자를 통해 더 세부적인 설정을 넣을 수 있다.
    context가 dispatcher나 ContinuationInterceptor를 갖고 있지 않으면 기본값인 Dispatchers.Default가 사용된다.
    부모 Job도 CoroutineScope에서 상속받지만, [context]로 재정의할 수 있다.
    
    기본적으로 코루틴은 즉시 실행되도록 스케줄링된다.
    [start] 인자를 통해 시작 옵션을 선택할 수 있다.
    [CoroutineStart.LAZY] 옵션을 선택하면 코루틴이 생성될 때 시작되지 않고,
    [Job.start]로 직접 시작하거나, [Job.join]이 호출될 때 암묵적으로 시작된다.
    
    코루틴에서 잡히지 않은 예외가 발생하면, 기본적으로 부모 Job이 취소된다.
    ([CoroutineExceptionHandler]를 따로 지정하지 않은 경우)
    즉, launch를 다른 코루틴 컨텍스트에서 사용중일 때 예외가 터지면 부모 코루틴도 취소된다.
    
    [newCoroutineContext]을 보면 새로 만든 코루틴의 디버깅 기능을 알 수 있다.
    
    @param
    context - [CoroutineScope.coroutineContext]에 추가로 붙일 코루틴 설정
    start - 코루틴 시작 옵션. [CoroutineStart.DEFAULT] 기본값
    block - 주어진 scope 내부에서 실행할 코루틴 코드 작성
    */
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

- `async`  살펴보기

    ```kotlin
    /*
    코루틴을 생성하고 미래의 결괏값을 [Deferred]객체로 반환한다.
    실행중인 코루틴은 [Deferred]가 [Job.cancel 호출 후 cancelled]취소되면 취소된다.
    
    다른 언어나 프레임워크의 async와 중요한 차이점 : 코루틴 async는 실패하면 부모 Job이나 외부 스코프를 취소시켜 구조적 동시성 원칙을 지킨다.
    이 동작을 바꾸고 싶다면, 감독하는 부모인 [SupervisorJob] 또는 [supervisorScope]를 사용할 수 있다.
    
    CoroutineContext는 [CoroutineScope]로부터 상속받는다.
    [context] 인자를 통해 더 세부적인 설정을 넣을 수 있다.
    context가 dispatcher나 ContinuationInterceptor를 갖고 있지 않으면 기본값인 Dispatchers.Default가 사용된다.
    부모 Job도 CoroutineScope에서 상속받지만, [context]로 재정의할 수 있다.
    
    기본적으로 코루틴은 즉시 실행되도록 스케줄링된다.
    [start] 인자를 통해 시작 옵션을 선택할 수 있다.
    [CoroutineStart.LAZY] 옵션을 선택하면 코루틴이 생성될 때 시작되지 않고 대기한다.
    [Job.start]로 직접 시작하거나, [Job.join], [Deferred.await]이 호출될 때 암묵적으로 시작된다.
    
    @param
    block - 주어진 scope 내부에서 실행할 코루틴 코드 작성
    */
    public fun <T> CoroutineScope.async(
        context: CoroutineContext = EmptyCoroutineContext,
        start: CoroutineStart = CoroutineStart.DEFAULT,
        block: suspend CoroutineScope.() -> T
    ): Deferred<T> {
        val newContext = newCoroutineContext(context)
        val coroutine = if (start.isLazy)
            LazyDeferredCoroutine(newContext, block) else
            DeferredCoroutine<T>(newContext, active = true)
        coroutine.start(start, coroutine, block)
        return coroutine
    }
    ```

- `newCoroutineContext` 살펴보기

    ```kotlin
    /**
    새 코루틴에 대한 컨텍스트를 생성한다.
    다른 디스패처나 ContinuationInterceptor가 지정되지 않은 경우 [Dispatchers.Default]로 지정된다.
     */
    @ExperimentalCoroutinesApi
    public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
        val combined = foldCopies(coroutineContext, context, true)
        val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
        return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
            debug + Dispatchers.Default else debug
    }
    ```

- `StandaloneCoroutine` 와 `LazyStandaloneCoroutine` 살펴보기

  `block.reateCoroutineUnintercepted` 뭐 차단해두고 하고 뭐그런거아닐까

  continuation.startCoroutineCancellable(this) 이걸로 시작하는건가

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
    
    /*
    active = false로 생성될 때 실행되지 않음
    onStart()가 호출되면 startCoroutineCancellable를 사용해 실행됨
    */
    private class LazyStandaloneCoroutine(
        parentContext: CoroutineContext,
        block: suspend CoroutineScope.() -> Unit
    ) : StandaloneCoroutine(parentContext, active = false) {
        private val continuation = block.createCoroutineUnintercepted(this, this)
    
        override fun onStart() {
        // startCoroutineCancellable는 실행되기전 취소될 수 있는 방식
            continuation.startCoroutineCancellable(this)
        }
    }
    ```

    - DeferredCoroutine과 LazyDeferredCoroutine Standalone과 매우 유사함

        ```kotlin
        private open class DeferredCoroutine<T>(
            parentContext: CoroutineContext,
            active: Boolean
        ) : AbstractCoroutine<T>(parentContext, true, active = active), Deferred<T> {
            override fun getCompleted(): T = getCompletedInternal() as T
            override suspend fun await(): T = awaitInternal() as T
            override val onAwait: SelectClause1<T> get() = onAwaitInternal as SelectClause1<T>
        }
        
        private class LazyDeferredCoroutine<T>(
            parentContext: CoroutineContext,
            block: suspend CoroutineScope.() -> T
        ) : DeferredCoroutine<T>(parentContext, active = false) {
            private val continuation = block.createCoroutineUnintercepted(this, this)
        
            override fun onStart() {
                continuation.startCoroutineCancellable(this)
            }
        }
        ```


### 차이점

|  | launch | async |
| --- | --- | --- |
| hasResult | false | true |
| return | Job | Deferred<T> |
| coroutine | StandaloneCoroutine | DeferredCoroutine |

## async 결괏값 받기

- Deferred는 Job과 같이 코루틴을 추상화한 객체이다.
- return 값을 결과로 반환하는 코루틴을 만들어낸다.

### Deferred

- Deferred 객체는 미래에 결괏값이 반환될 수 있음을 표현하는 코루틴 객체이다.
    - 코루틴이 실행완료될 때 결괏값이 반환된다.
    - 언제 결괏값이 반환될지 정확히 알 수 없다.
    - 결괏값이 필요하다면 수신될 때까지 대기해야한다.

*defer : 미루다, 연기하다

### await

- 결괏값 수신을 대기하는 함수
    - Deferred 코루틴이 실행 완료될 때까지 await 함수를 호출한 코루틴을 일시 중단한다.
    - Deferred 코루틴이 실행 완료되면 결괏값을 반환하고 호출부의 코루틴을 재개한다.
    - Job.join 함수와 매우 유사하게 동작한다.

*await : ~를 기다리다.

```kotlin
fun main() = runBlocking<Unit> {
    val networkDeferred: Deferred<String> = async(Dispatchers.IO) {
        delay(1000L)
        return@async "Dummy Response"
    }
   
    val result: String = networkDeferred.await()
    println(result) // Dummy Response
}
```

```kotlin
public suspend fun await(): T
```

## 여러개의 async로부터 결괏값 받기

- await 함수는 호출시 결괏값이 반환될 때까지 호출부의 코루틴이 일시 중단된다.
    - 아래 예제에서 `participantDeferred1.await()` 이 호출되면 호출부의 코루틴인 runBlocking 코루틴이 결괏값이 반환될 때까지 일시중단된다.
    - `participantDeferred2` 코루틴이 생성되기 전에 일시중단되어 동시에 처리할 수 있음에도 비효율적이게 순차적으로 처리된다. 소요시간 2초+

        ```kotlin
        fun main() = runBlocking<Unit> {
            val startTime = systemCurrentTime()
        
            val participantDeferred1: Deferred<Array<String>> = async(Dispatchers.IO) {
                delay(1000L)
                return@async arrayOf("James", "Jason")
            }
            val participants1: Array<String> = participantDeferred1.await()
        
            val participantDeferred2: Deferred<Array<String>> = async(Dispatchers.IO) {
                delay(1000L)
                return@async arrayOf("Jenny")
            }
            val participants2: Array<String> = participantDeferred2.await()
        
            println("[${getElapsedTime(startTime)}] 참여자 목록: ${listOf(*participants1, *participants2)}")
        }
        // output
        [지난 시간: 2022ms] 참여자 목록: [James, Jason, Jenny]
        ```

    - participantDeferred1, 2 코루틴 생성 후 await 호출 하기
    - `participantDeferred1.await()` 이 호출되면 runBlocking 코루틴이 결괏값을 반환할 때까지 일시중단되는 것은 같지만,

      중단 전에 participantDeferred2 코루틴 또한 동시에 실행됐기 때문에 소요시간은 1초+ 이다.

        ```kotlin
        fun main() = runBlocking<Unit> {
            val startTime = systemCurrentTime()
        
            val participantDeferred1: Deferred<Array<String>> = async(Dispatchers.IO) {
                delay(1000L)
                return@async arrayOf("James", "Jason")
            }
        
            val participantDeferred2: Deferred<Array<String>> = async(Dispatchers.IO) {
                delay(1000L)
                return@async arrayOf("Jenny")
            }
        
            val participants1: Array<String> = participantDeferred1.await()
            val participants2: Array<String> = participantDeferred2.await()
        
            println("[${getElapsedTime(startTime)}] 참여자 목록: ${listOf(*participants1, *participants2)}")
        }
        // output
        [지난 시간: 1013ms] 참여자 목록: [James, Jason, Jenny]
        ```


- 독립적인 각 코루틴들이 동시에 실행될 수 있도록 만드는 것은 성능 측면에서 매우 중요하다.

### awaitAll

- 여러개의 Deferred 객체로부터 결괏값을 받기 위한 함수
    - vararg(가변인자)로 Deferred 타입의 객체를 받고, 인자로 받은 모든 Deferred 코루틴으로부터 결과를 받을 때까지 호출부의 코루틴을 일시 중단한다.
    - 결과가 모두 수신되면 결괏값들을 List로 만들어 반환하고 호출부의 코루틴(runBlocking)을 재개한다.

    ```kotlin
    fun main() = runBlocking<Unit> {
        val startTime = systemCurrentTime()
    
        val participantDeferred1: Deferred<Array<String>> = async(Dispatchers.IO) {
            delay(1000L)
            return@async arrayOf("James", "Jason")
        }
    
        val participantDeferred2: Deferred<Array<String>> = async(Dispatchers.IO) {
            delay(1000L)
            return@async arrayOf("Jenny")
        }
    
        val results: List<Array<String>> = awaitAll(participantDeferred1, participantDeferred2)
        
        println("[${getElapsedTime(startTime)}] 참여자 목록: ${listOf(*results[0], *results[1])}")
    }
    // output
    [지난 시간: 1013ms] 참여자 목록: [James, Jason, Jenny]
    ```

  컬렉션에 대해 awaitAll 사용하기

    - Collection에 속한 Deferred이 모두 완료되어 결괏값을 반환할 때까지 호출부의 코루틴을 일시 중단한다.

    ```kotlin
    // results 부분만 변경 나머진 동일 [awaitAll(vararg)과 완전히 같게 동작한다]
    val results: List<Array<String>> = listOf(participantDeferred1, participantDeferred2).awaitAll()
    ```


```kotlin
/**
이 함수는 deferreds.map { it.await() }과 동등하지 않다.
 */
public suspend fun <T> awaitAll(vararg deferreds: Deferred<T>): List<T> =
    if (deferreds.isEmpty()) emptyList() else AwaitAll(deferreds).await()
    
public suspend fun <T> Collection<Deferred<T>>.awaitAll(): List<T> =
    if (isEmpty()) emptyList() else AwaitAll(toTypedArray()).await()
```

## Deferred는 Job + 결괏값

- 모든 코루틴 빌더는 Job 객체를 생성한다.
- Deferred 는 Job의 서브타입이다. → Job객체의 모든 함수와 프로퍼티 사용가능
- 코루틴으로부터 결괏값을 반환받는 await 기능이 추가됐다.

- Deferred 내부구현

    ```kotlin
    Deferred는 
    
    public interface Deferred<out T> : Job {
        public suspend fun await(): T
        public val onAwait: SelectClause1<T>
        
        // invokeOnCompletion 핸들러에서 사용
        public fun getCompleted(): T
        public fun getCompletionExceptionOrNull(): Throwable?
    }
    ```

- Job 내부구현

    ```kotlin
    public interface Job : CoroutineContext.Element {
        public companion object Key : CoroutineContext.Key<Job>
        public val parent: Job?
        public val isActive: Boolean
        public val isCompleted: Boolean
        public val isCancelled: Boolean
        public fun getCancellationException(): CancellationException
        public fun start(): Boolean
        public fun cancel(cause: CancellationException? = null)
        public val children: Sequence<Job>
        public fun attachChild(child: ChildJob): ChildHandle
        public suspend fun join()
        public val onJoin: SelectClause0
        public fun invokeOnCompletion(handler: CompletionHandler): DisposableHandle
        public fun invokeOnCompletion(
            onCancelling: Boolean = false,
            invokeImmediately: Boolean = true,
            handler: CompletionHandler): DisposableHandle
    ```


## withContext로 CoroutineContext 전환하기

- context 인자로 받은 CoroutineContext 객체를 사용해
- CoroutineScope인 block 람다식을 실행한다.
- 실행 완료되면 결과를 반환한다.
- withContext 함수의 block을 벗어나면 다시 기존의 CoroutineContext 객체를 사용해 코루틴을 재개한다.

### `async-await` 을 withContext로 대체하기

- Deferred객체 생성이 없어지고 "Dummy Response"를 바로 반환한다.

```kotlin
fun main() = runBlocking<Unit> {
    val result: String = withContext(Dispatchers.IO) {
        delay(1000L)
        return@withContext "Dummy Response"
    }
    print(result)
}
// output
Dummy Response
```

### withContext 동작방식

async-await은 새로운 코루틴을 생성해 작업을 처리한다.

- withContext는 실행중인 코루틴을 그대로 유지하고, 해당 코루틴의 실행 환경만 변경해 작업을 처리한다.
    - coroutine#1는 유지되고, 실행 환경이 context인자 값으로 변경된다(이를 컨텍스트 스위칭이라 부름).
    - withContext 함수의 context인자에 CoroutineDispatcher 객체를 할당해 사용하면 코루틴이 실행되는 데 사용할 CoroutineDispatcher객체를 자유롭게 바꿀수 있다(→ 실행될 스레드를 자유롭게 전환가능).

    ```kotlin
    fun main() = runBlocking<Unit> {
        printlnCurrentThreadNameWithText("runBlocking 블록1 실행")
    
        // withContext 스위칭1
        withContext(context = Dispatchers.IO) {
            printlnCurrentThreadNameWithText("withContext IO 블록 실행")
    
            // withContext 스위칭2
            withContext(context = this@runBlocking.coroutineContext) {
                printlnCurrentThreadNameWithText("withContext runBlockingContext 블록 실행")
    
                // withContext 스위칭3
                withContext(context = Dispatchers.Default) {
                    printlnCurrentThreadNameWithText("withContext Default 블록 실행")
                }
                printlnCurrentThreadNameWithText("withContext runBlocking 블록 마지막")
            }
            printlnCurrentThreadNameWithText("withContext IO 블록 마지막")
        }
        printlnCurrentThreadNameWithText("runBlocking 블록2 실행")
    }
    
    // output
    [main @coroutine#1] runBlocking 블록1 실행
    [DefaultDispatcher-worker-1 @coroutine#1] withContext IO 블록 실행
    [main @coroutine#1] withContext runBlockingContext 블록 실행
    [DefaultDispatcher-worker-1 @coroutine#1] withContext Default 블록 실행
    [main @coroutine#1] withContext runBlocking 블록 마지막
    [DefaultDispatcher-worker-1 @coroutine#1] withContext IO 블록 마지막
    [main @coroutine#1] runBlocking 블록2 실행
    ```

    - async-await과 비교하기
    - async함수는 새 코루틴을 생성하기 때문에 coroutine#1과 다른 coroutine#2가 생성됨

    ```kotlin
    fun main() = runBlocking<Unit> {
        printlnCurrentThreadNameWithText("runBlocking 블록1 실행")
        async(Dispatchers.IO) {
            printlnCurrentThreadNameWithText("async 블록 실행")
        }.await()
        printlnCurrentThreadNameWithText("runBlocking 블록2 실행")
    }
    // output
    [main @coroutine#1] runBlocking 블록1 실행
    [DefaultDispatcher-worker-1 @coroutine#2] async 블록 실행
    [main @coroutine#1] runBlocking 블록2 실행
    ```


### withContext 주의사항

- withContext는 새로운 코루틴을 만들지 않기 때문에
- withContext함수가 여러번 호출되면 동시에 실행되지 않고 순차적으로 실행된다.
- 여러 작업이 병렬로 실행되어야 하는 상황에서 문제가 생김
    - withContext를 실행할 때 새로운 코루틴이 생성되지 않음
    - runBlocking 코루틴을 main → IO 로 컨텍스트 스위칭

    ```kotlin
    fun main() = runBlocking<Unit> {
        val startTime = systemCurrentTime()
        val helloString: String = withContext(Dispatchers.IO) {
            delay(1000L)
            return@withContext "Hello"
        }
    
        val worldString: String = withContext(Dispatchers.IO) {
            delay(1000L)
            return@withContext "World"
        }
    
        println("[${getElapsedTime(startTime)}] $helloString $worldString")
    }
    // output
    [지난 시간: 2022ms] Hello World
    ```

    - async-await을 사용하면 이전에 봤듯이 새로운 코루틴을 생성해 동시에 실행됨

- withContext 살펴보기

    ```kotlin
    public suspend fun <T> withContext(
        context: CoroutineContext,
        block: suspend CoroutineScope.() -> T
    ): T {
        contract {
            callsInPlace(block, InvocationKind.EXACTLY_ONCE)
        }
        return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
            val oldContext = uCont.context
            val newContext = oldContext.newCoroutineContext(context)
            newContext.ensureActive()
            if (newContext === oldContext) {
                val coroutine = ScopeCoroutine(newContext, uCont)
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
            if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
                val coroutine = UndispatchedCoroutine(newContext, uCont)
                withCoroutineContext(coroutine.context, null) {
                    return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
                }
            }
            val coroutine = DispatchedCoroutine(newContext, uCont)
            block.startCoroutineCancellable(coroutine, coroutine)
            coroutine.getResult()
        }
    }
    ```