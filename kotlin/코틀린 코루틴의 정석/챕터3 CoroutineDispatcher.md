# CoroutineDispatcher

Dispatch : (특히 특별한 목적을 위해) 보내다

-er : 무엇을 하는 사람이나 도구, 기계 등

CoroutineDispatcher : 코루틴을 보내는 도구

## CoroutineDispatcher는 Coroutine을 스레드풀로 보낸다.

CoroutineDispatcher는 스레드풀 또는 스레드로 코루틴을 보낸다.

코루틴은 일시 중단 가능한 ‘작업’이며, 작업은 스레드가 있어야 실행될 수 있다.

## CoroutineDispatcher는 Coroutine 작업대기열을 관리한다.

CoroutineDispatcher는 작업대기열을 가진다.

작업대기열은 실행되어야할 코루틴 작업들이 적재된 곳

CoroutineDispatcher객체는 자신이 사용가능한 스레드가 있는지 확인하고

→ 있다면 스레드로 보내 실행시킨다.

→ 없다면 대기

## 스레드풀이 제한된 CoroutineDispatcher

~~(무제한 디스패처는 챕터 11에서 다룬다.)~~

CoroutineDispatcher 객체별로 어떤 작업을 처리하는 디스패처인지 미리 역할을 정해두고 실행하는 것이 효율적이다.

제한된 디스패처는 스레드풀 또는 스레드가 제한된 CoroutineDispatcher 객체를 의미한다.

스레드의 개수와 스레드 이름을 지정한다.

n > 2의 경우 스레드이름은 name-1, name-2 으로 표시된다.

```kotlin
    val singleThreadCoroutineDispatcher = newSingleThreadContext(name = "SingleThread")

    val multiThreadCoroutineDispatcher = newFixedThreadPoolContext(
        nThreads = 3,
        name = "multiThread"
    )

// output
Running, pool size = 2, active threads = 1, queued tasks = 0, completed tasks = 1
```

## CoroutineDispatcher는 자식 코루틴으로 상속가능하다

코루틴은 내부에 새로운 코루틴을 실행해 부모-자식 구조를 가질 수 있다.

바깥쪽 코루틴을 부모 코루틴, 내부 코루틴을 자식 코루틴이라고 한다.

- 부모 코루틴의 실행환경을 자식 코루틴에게 전달 할 수 있다.
    - 자식 코루틴의 CoroutineDispatcher가 설정되지 않았으면 → 부모 CoroutineDispatcher를 사용한다.

```kotlin
fun main() = runBlocking<Unit> {
    val multiThreadCoroutineDispatcher = newFixedThreadPoolContext(
        nThreads = 2,
        name = "multiThread"
    )

    launch(context = multiThreadCoroutineDispatcher) {
        currentThreadNameLogWithText("부모")
        launch(context = multiThreadCoroutineDispatcher) {
            currentThreadNameLogWithText("자식")
        }
    }
}

// output
[multiThread-1] 부모
[multiThread-2] 자식
```

## 미리 정의된 CoroutineDispatcher

newFixedThreadPoolContext를 사용하면 DelicateCoroutinesApi 경고 문구가 뜬다.

- newFixedThreadPoolContext를 사용해 직접 CoroutineDispatcher 객체를 만드는 것은 비효율적일 가능성이 높다.
    - 특정 CoroutineDispatcher 객체에서만 사용되는 스레드풀이 생성됨
    - 스레드풀에 속한 스레드의수가 비효율적일 가능성 (너무 많거나, 너무 적거나)
    - 스레드의 생성 비용은 비싸다.

→ 코루틴 라이브러리에서 미리 정의된 CoroutineDispatcher

멀티 스레드 프로그래밍이 필요한 일반적인 상황에 맞춰 만들어졌기 때문에 제공되는 CoroutineDispatcher를 사용해 코루틴을 실행하자

### `Dispatcher.IO`  → 네트워크 요청, 파일 입출력 등 I/O 작업을 위한

- 멀티 스레드 프로그래밍이 가장 많이 사용되는 작업 (HTTP 요청, DB 입출력 등)
- 스레드 수 : JVM에서 사용 가능한 프로세서 수, 64 중 큰값 (1.10.1 기준)

### `Dispatcher.Default` → CPU를 많이 사용하는 연산 작업을 위한

- CPU 연산이 필요한 대용량 데이터 처리 작업

### `Dispatcher.Main` → 메인 스레드를 사용하기 위한

- UI가 있는 앱에서 메인 스레드를 사용

`~~Dispatcher.Unconfined` (무제한 디스패처, 11장에서 다룬다.)~~

### 공유 스레드풀

코루틴이 실행된 스레드의 이름이 DefaultDispatcher-worker-n 이다.

DefaultDispatcher-worker는 코루틴 라이브러리에서 제공하는 공유 스레드풀에 속한 스레드이다.

`Dispatcher.IO` , `Dispatcher.Default` 는 공유 스레드풀의 스레드를 사용할 수 있다.

- 스레드의 생성과 관리를 효율적
- 공유 스레드풀에서 스레드를 무제한 생성 및 사용 가능
- `Dispatcher.IO` 와 `Dispatcher.Default` 가 사용하는 스레드는 구분 된다. [107p 그림3-15 참고]
- 쓸 수 있는 개수가 나눠져있는 것 뿐이지 스레드는 공유해서 사용된다.

newFixedThreadPoolContext로 생성된 Dispatcher는 자신만 사용할 수 있는 전용 스레드풀을 생성하는 것이다.

```kotlin
fun main() = runBlocking<Unit> {
    launch(context = Dispatchers.IO) {
        currentThreadNameLogWithText("아이오")
    }
    launch(context = Dispatchers.Default) {
        currentThreadNameLogWithText("디폴트")
    }
}

// output
[DefaultDispatcher-worker-2] 디폴트
[DefaultDispatcher-worker-1] 아이오
```

### `IO` vs `Default` 차이

- 입출력 작업은 실행한 뒤 결과를 반환받을 때까지 스레드를 사용하지 않는다.
    - 스레드가 대기하는 동안 다른 작업을 동시에 실행

  → 스레드 기반 작업보다 코루틴 속도가 빠름

- CPU 바운드 작업은 작업동안 지속적으로 스레드를 사용한다.

  → 스레드 기반 작업과 코루틴 속도 비슷


### 제한된 병렬처리 limitedParallelism

`Dispatcher.Default`를 사용해 무겁고 오래걸리는 연산을 처리하면, 해당 연산을 위해 `Dispatcher.Default`의 모든 스레드가 사용될 수 있다.

→ 다른 연산이 실행되지 못하는 현상 발생 가능

사용할 스레드의 수를 제한 할 수 있는 limitedParallelism 함수 제공

[109p 그림 3-16 확인]

`Dispatcher.Default.limitedParallelism()` 의 경우는 Dispatcher.Default가 사용할 수 있는 스레드의 일부를 사용하는 것이다.

`Dispatcher.IO.limitedParallelism()` → 공유 스레드 풀의 스레드로 구성된 새로운 스레드 풀을 만들어 사용. + 스레드의 수 제한 없이 생성가능

특정 작업이 다른 작업에 영향을 받지 않고 별도의 스레드에서 실행이 필요할 때

공유 스레드풀에서 새로운 스레드를 만들어냄 → 비싼 작업 남용 금지

```kotlin
fun main() = runBlocking<Unit> {
    val dispatcher = Dispatchers.Default.limitedParallelism(2)
    launch(context = dispatcher) {
        repeat(1) {
            printlnCurrentThreadNameWithText("제한된 디폴트1 ${it + 1}")
        }
    }
    launch(context = dispatcher) {
        repeat(1) {
            printlnCurrentThreadNameWithText("제한된 디폴트2 ${it + 1}")
        }
    }
    launch(context = dispatcher) {
        repeat(1) {
            printlnCurrentThreadNameWithText("제한된 디폴트3 ${it + 1}")
        }
    }
}

// output no limit
[DefaultDispatcher-worker-2] 제한된 디폴트2 1
[DefaultDispatcher-worker-1] 제한된 디폴트1 1
[DefaultDispatcher-worker-3] 제한된 디폴트3 1

// ouput limit 1 
[DefaultDispatcher-worker-1] 제한된 디폴트1 1
[DefaultDispatcher-worker-1] 제한된 디폴트2 1
[DefaultDispatcher-worker-1] 제한된 디폴트3 1

// ouput limit 2
[DefaultDispatcher-worker-2] 제한된 디폴트2 1
[DefaultDispatcher-worker-1] 제한된 디폴트1 1
[DefaultDispatcher-worker-2] 제한된 디폴트3 1

// ouput limit 3 + n (n > 3)
[DefaultDispatcher-worker-2] 제한된 디폴트2 1
[DefaultDispatcher-worker-1] 제한된 디폴트1 1
[DefaultDispatcher-worker-3] 제한된 디폴트3 1
```

## Thread.sleep vs delay

`공통점`

- 작업의 실행을 일정 시간 동안 지연시킴

`차이점`

- Thread.sleep →Thread 자체를 블로킹함 (다른 코루틴이 사용 불가능)
- delay → 해당 코루틴을 지연시간동안 Thread에서 탈착 (다른 코루틴이 사용 가능)

```kotlin
fun main() = runBlocking<Unit> {
    val multiThreadCoroutineDispatcher = newFixedThreadPoolContext(
        nThreads = 2,
        name = "multiThread"
    )

    launch(multiThreadCoroutineDispatcher) { // [Running, pool size = 1, active threads = 1, queued tasks = 0, completed tasks = 1]
        Thread.sleep(1000)
        currentThreadNameLogWithText("실행1")
    }
    launch(multiThreadCoroutineDispatcher) {
        Thread.sleep(1000)
        currentThreadNameLogWithText("실행2")
    }
    launch(multiThreadCoroutineDispatcher) {
        Thread.sleep(1000)
        currentThreadNameLogWithText("실행3")
    }
}

// print
<1초 후>
[multiThread-2] 실행2
[multiThread-1] 실행1
<1초 후>
[multiThread-2] 실행3
```

```kotlin
fun main() = runBlocking<Unit> {
    val multiThreadCoroutineDispatcher = newFixedThreadPoolContext(
        nThreads = 2,
        name = "multiThread"
    )

    launch(multiThreadCoroutineDispatcher) { // [Running, pool size = 1, active threads = 1, queued tasks = 0, completed tasks = 1]
        delay(1000)
        currentThreadNameLogWithText("실행1")
    }
    launch(multiThreadCoroutineDispatcher) {
        delay(1000)
        currentThreadNameLogWithText("실행2")
    }
    launch(multiThreadCoroutineDispatcher) {
        delay(1000)
        currentThreadNameLogWithText("실행3")
    }
}

// print
<1초 후>
[multiThread-2] 실행1
[multiThread-1] 실행2
[multiThread-1] 실행3
```