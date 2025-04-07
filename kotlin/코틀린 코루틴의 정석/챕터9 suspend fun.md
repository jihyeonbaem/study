# 일시 중단 함수 (suspend + fun)

일시 중단 함수는 → 함수 내에서 일시 중단 지점을 포함할 수 있다.

코루틴의 비동기 작업과 관련된 복잡한 코드들을 구조화하고 재사용할 수 있는 코드의 집합으로 만드는 데 사용된다.

일시 중단 함수(suspend fun) → 일반 함수(fun) + 일시 중단 지점 포함 가능

## suspend fun의 기능

fun의 기능 : 재사용 → 불필요한 중복 제거

suspend의 기능 : 일시 중단 지점 포함

```kotlin
fun main() = runBlocking<Unit> {
    delay(1000L)
    println("Hello World!")
    delay(1000L)
    println("Hello World!")
}
```

```kotlin
fun main() = runBlocking<Unit> {
    delayAndPrintHelloWorld()
    delayAndPrintHelloWorld()
}

suspend fun delayAndPrintHelloWorld() {
    delay(1000L)
    println("Hello World!")
}
```

> *일반 함수에서 suspend fun을 호출하려고 할 때 발생하는 에러
>
>
> `Suspend function 'suspend fun delay(timeMillis: Long): Unit' should be called only from a coroutine or another suspend function.`
>
> → suspend fun은 코루틴이나 다른 suspend fun에서만 호출될 수 있다.
>

## suspend fun은 Coroutine이 아니다

그냥, 코루틴 내부에서 실행되는 함수(코드 집합)일 뿐이다.

- runBlocking 코루틴 1개 생성
- delayAndPrintHelloWorld() 순차적 실행 → 총 2초 지연

```kotlin
fun main() = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()
    delayAndPrintHelloWorld()
    delayAndPrintHelloWorld()
    println(getElapsedTime(startTime))
}
// output
Hello World!
Hello World!
지난 시간: 2015ms
```

```kotlin
fun main() = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()
    delay(1000L)
    println("Hello World!")
    delay(1000L)
    println("Hello World!")
    println(getElapsedTime(startTime))
}
```

→ 코루틴을 생성하는 코드가 없음 (runBlocking 제외)

- main 함수를 runBlocking 없이 suspend로 만든다면?

    ```kotlin
    fun main() {
        printlnCurrentThreadNameWithText("hello")
    }
    // output
    [main] hello
    ```

  → 코루틴이 없는 코드

    ```kotlin
    suspend fun main() {
        delay(100L)
        printlnCurrentThreadNameWithText("hello")
    }
    // output
    [kotlinx.coroutines.DefaultExecutor] hello
    ```

  → ?? 이건 뭐지

    ```kotlin
    suspend fun main() {
        printlnCurrentThreadNameWithText("hello")
        delay(100L)
        printlnCurrentThreadNameWithText("hello")
    }
    // output
    [main] hello
    [kotlinx.coroutines.DefaultExecutor] hello
    ```

  → GPT피셜 : suspend fun main()을 실행하기 위해서 백그라운드에서 runBlocking을 감싸서 실행해준다. (runBlocking { main() } 형태) → 코루틴 내부에서 실행됨

  *DefaultExecutor → delay() 호출시 코루틴이 중단되었다가 재개될때 스케줄러인 `DefaultExecutor` 스레드에서 실행될 수 있다.


### suspend fun을 별도의 코루틴에서 실행하기

```kotlin
fun main() = runBlocking<Unit> {
    val startTime = System.currentTimeMillis()
    launch {
        delayAndPrintHelloWorld()
    }
    launch {
        delayAndPrintHelloWorld()
    }
    println(getElapsedTime(startTime))
}
// output
지난 시간: 2ms // delay로 main 스레드를 사용권한을 양보받아 즉시 실행됨
Hello World!
Hello World!
```

## suspend fun 사용하기

일시 중단 지점을 포함할 수 있기 때문에 → 일시 중단을 할 수 있는 곳에서만 호출할 수 있다.

`Suspend function should be called only from a coroutine or another suspend function.`

### 일시 중단이 가능한 지점

1. 코루틴 내부

    ```kotlin
    fun main() = runBlocking<Unit> {
        delayAndPrint("Parent Coroutine")
        launch {
            delayAndPrint("Child Coroutine")
        }
    }
    ```

2. 일시 중단 함수

    ```kotlin
    suspend fun searchByKeyword(keyWorld: String): Array<String> {
        val dbResults = searchFromDB(keyWorld)
        val serverResults = searchFromServer(keyWorld)
    
        return arrayOf(*dbResults, *serverResults)
    }
    
    suspend fun searchFromDB(keyWorld: String): Array<String> {
        delay(1000L)
        return arrayOf("[DB]${keyWorld}1", "[DB]${keyWorld}2")
    }
    
    suspend fun searchFromServer(keyWorld: String): Array<String> {
        delay(1000L)
        return arrayOf("[Server]${keyWorld}1", "[Server]${keyWorld}2")
    }
    ```


### suspend fun 내부에서 코루틴을 생성하려면

searchFromDB, searchFromServer가 순차적으로 실행 되어 총 2초가 걸린다.

순차적으로 실행되지 않도록 하기 위해서 async 코루틴 빌더 함수를 사용해 서로다른 코루틴에서 실행되도록 만들어보자

```kotlin
fun <T> CoroutineScope.async( ... )
```

`async` 코루틴 빌더 함수는 CoroutineScope의 확장 함수이다.

→ suspend fun 내부에서는 suspend fun을 호출한 코루틴의 CoroutineScope 객체에 접근할 수 없다.

### coroutineScope을 사용해  suspend fun 내부에서 코루틴 실행하기

suspend fun 내부에서 CoroutineScope 객체에 접근할 수 있도록 만들어 줘야 한다.

`coroutineScope` 함수를 사용해 새로운 CoroutineScope 객체를 생성한다.

1. 구조화를 깨지 않는다.
2. 생성된 CoroutineScope 객체는 block 람다에서 this로 접근 가능

```kotlin
public suspend fun <R> coroutineScope(block: suspend CoroutineScope.() -> R): R
```

- 서로 다른 코루틴에서 동시에 실행됨 → 총 1초 소요

```kotlin
suspend fun searchByKeyword(keyWorld: String): Array<String> = coroutineScope {
    val dbResults = async { searchFromDB(keyWorld) }
    val serverResults = async { searchFromServer(keyWorld) }

    return@coroutineScope arrayOf(*dbResults.await(), *serverResults.await())
}
```

```mermaid
flowchart TB
A[runBlocking**]
B[coroutineScope]
C[dbResults]**
D**[serverResults]**

A --> B --> C & D

부모 ---> 자식
  
  %% 스타일 설정
	classDef default font-size:13px,stroke-width:0px,color:#111;
```

만약, dbReusults에서 예외가 발생 하면 → 부모 코루틴인 coroutineScope(+ runBlocking)까지 예외가 전파되고 코루틴이 취소되어 → coroutineScope의 자식 코루틴인 serverResults 또한 취소된다.

### supervisorScope를 사용해 suspend fun 내부에서 생성된 코루틴의 예외 전파를 제한하기

supervisorScope는 예외 전파를 제한한다.

CoroutineScope 객체 하위에서 실행되는 자식 코루틴들의 예외 전파를 제한한다.

supervisorScope = coroutineScope + SupervisorJob (Job대신 생성됨)

supervisorScope을 사용하면 `dbResultsDeferred`에서 발생한 예외가 부모코루틴인 supervisorScope로 전파되지 않는다.

Deferred 객체는 await 함수 호출시 예외를 추가로 노출하므로 try-catch문으로 감싸 예외발생시 emptyArray가 반환되도록한다.

```kotlin
suspend fun searchByKeyword(keyWorld: String): Array<String> = supervisorScope {
    val dbResultsDeferred = async {
        throw Exception("dbResults에서 예외 발생")
        searchFromDB(keyWorld)
    }
    val serverResultsDeferred = async {
        searchFromServer(keyWorld)
    }

    val dbResults = try {
        dbResultsDeferred.await()
    } catch (e: Exception) {
        emptyArray<String>()
    }

    val serverResults = try {
        serverResultsDeferred.await()
    } catch (e: Exception) {
        emptyArray<String>()
    }

    return@supervisorScope arrayOf(*dbResults, *serverResults)
}
```