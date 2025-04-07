## CoroutineContext 구성요소

- CoroutineContext는 코루틴의 실행환경을 설정하고 관리하는 인터페이스이다.
- 코루틴의 실행과 관련된 모든 설정은 CoroutineContext 객체를 통해 이뤄진다.

`코루틴을 실행시키는 핵심 요소`

- CoroutineName : 코루틴의 이름 설정한다.
- CoroutineDispatcher : 코루틴을 스레드에 할당해 실행한다.
- Job : 코루틴의 추상체, 코루틴을 조작하는 데 사용된다.
- CoroutineExceptionHandler : 코루틴에서 발생한 예외를 처리한다.

이외에도 다양한 CoroutineContext 구성요소가 있다.

## CoroutineContext 구성하기

- CoroutineContext 객체는 Key-Value 쌍으로 각 구성 요소를 관리한다.
- 각 구성 요소는 고유한 Key를 가지고, Key는 중복된 Value를 허용하지 않는다.
    - CoroutineContext 객체는 CoroutineName, CoroutineDispathcer, Job, CoroutineExceptionHandler 객체를 한 개씩만 가질 수 있다.

  | Key | Value |
      | --- | --- |
  | CoroutineName 키 | CoroutineName 객체 |
  | CoroutineDispatcher 키 | CoroutineDispatcher 객체 |
  | Job 키 | Job 객체 |
  | CoroutineExceptionHandler 키 | CoroutineExceptionHandler 객체 |
- CoroutineContext 객체는 `+`연산자를 사용해 구성된다.


    | Key | Value |
    | --- | --- |
    | CoroutineName 키 | CoroutineName("MyCoroutine") |
    | CoroutineDispatcher 키 | newSingleThreadContext("MyThread") |
    | Job 키 | null |
    | CoroutineExceptionHandler 키 | null |
    
    ```kotlin
    fun main() = runBlocking<Unit> {
        val coroutineContext: CoroutineContext = newSingleThreadContext("MyThread") + CoroutineName("MyCoroutine")
        println(coroutineContext[CoroutineName])
        println(coroutineContext[CoroutineDispatcher])
        println(coroutineContext[Job])
        println(coroutineContext[CoroutineExceptionHandler])
        
        launch(coroutineContext) {
            printlnCurrentThreadNameWithText("실행")
        }
    }
    // output
    CoroutineName(MyCoroutine)
    java.util.concurrent.ScheduledThreadPoolExecutor@59f95c5d[Running, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 0]
    null
    null
    [MyThread @MyCoroutine#2] 실행
    ```

- 구성요소가 없는 CoroutineContext 만들기

    ```kotlin
    fun main() = runBlocking<Unit> {
        val coroutineContext: CoroutineContext = EmptyCoroutineContext
        println(coroutineContext[CoroutineName])
        println(coroutineContext[CoroutineDispatcher])
        println(coroutineContext[Job])
        println(coroutineContext[CoroutineExceptionHandler])
        launch(coroutineContext) {
            printlnCurrentThreadNameWithText("실행")
        }
    }
    // output
    null
    null
    null
    null
    [main @coroutine#2] 실행
    ```


### CoroutineContext 구성요소 대체하기

- CoroutineContext 객체에 같은 Key를 가진 구성요소가 더해진다면, 나중에 추가된 구성요소가 이전값을 대체한다.
- 같은 구성 요소에 대해서 마지막에 들어온 하나의 값만 취한다.

    ```kotlin
    fun main() = runBlocking<Unit> {
        val coroutineContext: CoroutineContext = newSingleThreadContext("MyThread") + CoroutineName("MyCoroutine")
        val newCoroutineContext: CoroutineContext = coroutineContext + CoroutineName("NewCoroutine")
    
        launch(newCoroutineContext) {
            printlnCurrentThreadNameWithText("실행")
        }
    }
    // output
    [MyThread @NewCoroutine#2] 실행
    ```


### CoroutineContext 구성요소 합치기

- 위와 동일하게 동작한다.
- 같은 key를 가진 CoroutineDispatcher와 CoroutineName이 나중에 추가된 요소인 `coroutineContext2` 으로 대체된다.

    ```kotlin
    fun main() = runBlocking<Unit> {
        val coroutineContext1: CoroutineContext = newSingleThreadContext("MyThread1") + CoroutineName("MyCoroutine1")
        val coroutineContext2: CoroutineContext = newSingleThreadContext("MyThread2") + CoroutineName("MyCoroutine2")
    
        val combinedCoroutineContext: CoroutineContext = coroutineContext1 + coroutineContext2
    
        launch(combinedCoroutineContext) {
            printlnCurrentThreadNameWithText("실행")
        }
    }
    // output
    [MyThread2 @MyCoroutine2#2] 실행
    ```


### CoroutineContext에 Job 추가하기

Job객체는 launch나 runBlocking 같은 코루틴 빌더를 통해 자동으로 생성된다.

`Job()`을 호출해 생성할 수도 있다.

> 주의! Job 객체를 직접 생성해 추가하면 코루틴의 구조화가 깨진다.
>

```kotlin
fun main() = runBlocking<Unit> {
    val myJob = Job()
    val coroutineContext: CoroutineContext = Dispatchers.IO + myJob

    println(coroutineContext[CoroutineName])
    println(coroutineContext[CoroutineDispatcher])
    println(coroutineContext[Job])
    println(coroutineContext[CoroutineExceptionHandler])
}
// output
null
Dispatchers.IO
JobImpl{Active}@3d04a311
null
```

| Key | Value |
| --- | --- |
| CoroutineName 키 | null |
| CoroutineDispatcher 키 | Dispatchers.IO |
| Job 키 | myJob |
| CoroutineExceptionHandler 키 | null |

### CoroutineContext 구성요소 접근하기

- CoroutineContext 객체의 구성 요소에 접근하기 위해서는 각 구성 요소가 가진 고유한 Key가 필요하다.
- CoroutineContext 구성요소는 자신의 내부에 Key를 싱글톤 객체로 구현한다.
    - `CoroutineContext.Key<CoroutineName>` 를 구현하는 companion object `Key`

| CoroutineContext 구성요소 | CoroutineContext 구성요소 Key |
| --- | --- |
| CoroutineName | CoroutineName.Key |
| CoroutineDispatcher | CoroutineDispatcher.Key |
| Job | Job.Key |
| CoroutineExceptionHandler | CoroutineExceptionHandler.Key |
- CoroutineName 내부구현

    ```kotlin
    public data class CoroutineName(
        val name: String
    ) : AbstractCoroutineContextElement(CoroutineName) {
        public companion object Key : CoroutineContext.Key<CoroutineName>
    }
    ```

- CoroutineDispatcher 내부구현

    ```kotlin
    public abstract class CoroutineDispatcher : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
        public companion object Key : AbstractCoroutineContextKey<ContinuationInterceptor, CoroutineDispatcher>(
            ContinuationInterceptor,
            { it as? CoroutineDispatcher })
           
        ...
        
        override fun toString(): String = "$classSimpleName@$hexAddress"
    }
    ```

- Job 내부구현

    ```kotlin
    public interface Job : CoroutineContext.Element {
        public companion object Key : CoroutineContext.Key<Job>
        ...
    }
    ```

- CoroutineExceptionHandler 내부구현

    ```kotlin
    public interface CoroutineExceptionHandler : CoroutineContext.Element {
        public companion object Key : CoroutineContext.Key<CoroutineExceptionHandler>
    
        public fun handleException(context: CoroutineContext, exception: Throwable)
    }
    ```


CoroutineContext 구성요소의 Key는 고유해 모두 같은 값을 반환하다.

```kotlin
fun main() = runBlocking<Unit> {
    val dispatcherKey1: CoroutineContext.Key<*> = Dispatchers.IO.key
    val dispatcherKey2: CoroutineContext.Key<*> = Dispatchers.Default.key

    println(dispatcherKey1 === dispatcherKey2)
}
// output
true
```

- coroutineContext 변수에 operator fun인 get의 key 인자에 CoroutineName.Key를 넘겨

  → nameFromContext에서 CoroutineName 객체만 가져올 수 있다.

    ```kotlin
    fun main() = runBlocking<Unit> {
        val coroutineContext: CoroutineContext = Dispatchers.IO + CoroutineName("MyCoroutine")
        
        // val nameFromContext: CoroutineName? = coroutineContext.get(key = CoroutineName.Key)
        // get함수는 CoroutineContext의 operator 함수이다.
        val nameFromContext: CoroutineName? = coroutineContext[CoroutineName.Key]
    
        println(nameFromContext)
    }
    // output
    CoroutineName(MyCoroutine
    ```

- Key를 명시적으로 사용하지 않아도 구성요소 자체를 키로 사용할 수 있다.

  CoroutineContext 구성요소 객체는 companion object로 `CoroutineContext.Key<*>` 를 구현하는 Key를 갖고있다.

  Key가 들어갈 자리에 CoroutineName을 사용하면 자동으로 CoroutineName.Key를 사용해 연산을 처리한다. [간결한 코드]

    ```kotlin
    // 나머지는 위 예시코드와 동일
    val nameFromContext: CoroutineName? = coroutineContext[CoroutineName]
    ```


- CoroutineContext 구성요소 인스턴스의 key 프로퍼티를 사용해 접근하기

  해당 key 프로퍼티는 companion object로 선언된 Key와 동일한 객체를 가리킨다.

    ```kotlin
    fun main() = runBlocking<Unit> {
        val coroutineName: CoroutineName = CoroutineName("MyCoroutine")
        val coroutineDispatcher: CoroutineDispatcher = Dispatchers.IO
        val coroutineContext: CoroutineContext = coroutineName + coroutineDispatcher
    
        println(coroutineContext[coroutineName.key])
        println(coroutineContext[coroutineDispatcher.key])
        println(coroutineName.key === CoroutineName.Key)
    }
    // output
    CoroutineName(MyCoroutine)
    Dispatchers.IO
    true
    ```


### CoroutineContext 구성요소 제거하기

- CoroutineContext 객체는 구성 요소를 제거하기 위한 minusKey 함수를 제공한다.
    - minusKey 함수는 구성 요소의 Key를 인자로 받아 해당 구성 요소를 제거하고 나머지 CoroutineContext 객체를 반환한다.

    ```kotlin
    // 지정된 key를 제외한 CoroutineContext를 반환한다.
    public fun minusKey(key: Key<*>): CoroutineContext
    ```

- minusKey로 구성요소 제거하기
    - coroutineContext.minusKey의 key인자로 CoroutineName을 넘겨 해당 요소를 제거함

  > 주의! minusKey를 사용할 때 새로운 CoroutineContext객체가 반환된다.
  >
  >
  > minusKey를 호출한 CoroutineContext객체는 그대로 유지된다. (deletedCoroutineContext 이후 coroutineContext 출력확인)
  >

  `coroutineContext`

  | Key | Value |
      | --- | --- |
  | CoroutineName 키 | CoroutineName("MyCoroutine") |
  | CoroutineDispatcher 키 | Dispatchers.IO |
  | Job 키 | myJob |

  `deletedCoroutineContext`

  | Key | Value |
      | --- | --- |
  | CoroutineName 키 | null |
  | CoroutineDispatcher 키 | Dispatchers.IO |
  | Job 키 | myJob |

    ```kotlin
    fun main() = runBlocking<Unit> {
        val coroutineName: CoroutineName = CoroutineName("MyCoroutine")
        val coroutineDispatcher: CoroutineDispatcher = Dispatchers.IO
        val myJob: CompletableJob = Job()
        val coroutineContext: CoroutineContext = coroutineName + coroutineDispatcher + myJob
    
        println(coroutineContext)
    
        val deletedCoroutineContext: CoroutineContext = coroutineContext.minusKey(CoroutineName)
        println(deletedCoroutineContext)
        println(coroutineContext)
    }
    // output
    [CoroutineName(MyCoroutine), JobImpl{Active}@7a46a697, Dispatchers.IO]
    [JobImpl{Active}@7a46a697, Dispatchers.IO]
    [CoroutineName(MyCoroutine), JobImpl{Active}@7a46a697, Dispatchers.IO]
    ```


- CoroutineContext 내부구현

    ```kotlin
    public interface CoroutineContext {
        public operator fun <E : Element> get(key: Key<E>): E?
    
        public fun <R> fold(initial: R, operation: (R, Element) -> R): R
    
        public operator fun plus(context: CoroutineContext): CoroutineContext =
            if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation
                context.fold(this) { acc, element ->
                    val removed = acc.minusKey(element.key)
                    if (removed === EmptyCoroutineContext) element else {
                        val interceptor = removed[ContinuationInterceptor]
                        if (interceptor == null) CombinedContext(removed, element) else {
                            val left = removed.minusKey(ContinuationInterceptor)
                            if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                                CombinedContext(CombinedContext(left, element), interceptor)
                        }
                    }
                }
    		// 지정된 key를 제외한 CoroutineContext를 반환한다.
        public fun minusKey(key: Key<*>): CoroutineContext
    
    		// CoroutineContext 요소에 대한 Key. E는 key를 가진 요소의 유형이다.
        public interface Key<E : Element>
    
        public interface Element : CoroutineContext {
    
            public val key: Key<*>
    
            public override operator fun <E : Element> get(key: Key<E>): E? =
                if (this.key == key) this as E else null
    
            public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
                operation(initial, this)
    
            public override fun minusKey(key: Key<*>): CoroutineContext =
                if (this.key == key) EmptyCoroutineContext else this
        }
    }
    
    ```

    ```kotlin
    public abstract class AbstractCoroutineContextElement(public override val key: Key<*>) : Element
    ```