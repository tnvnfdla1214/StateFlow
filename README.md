# StateFlow

이포스터는 LiveData에서 StateFlow로 바꿔야하는 이유와 바꾸는 방법에 대해서 작성합니다.

이 포스터는 [이 블로그](https://proandroiddev.com/should-we-choose-kotlins-stateflow-or-sharedflow-to-substitute-for-android-s-livedata-2d69f2bd6fa5) 를 참고하여 작성하였습니다.

## LiveData란
LiveData 는 옵저버 패턴을 활용하여 구현되었으며, 관찰 가능한 일반 클래스인 ObservableXXX 클래스와는 달리 LiveData 는 생명주기의 변화를 인식합니다. 즉, Activity, Fragment, Service 등 안드로이드 컴포넌트의 생명 주기 인식을 통해 Active 상태에 있는 컴포넌트에서만 업데이트합니다. 자세한 내용은 [LiveData](https://github.com/tnvnfdla1214/LiveData)를 확인해주세요.

LiveData 객체는 주로 AAC ViewModel 에서 관리하게 되는데, 이 또한 Presentation Layer 에 속해 있으니 크게 문제가 될 것은 없어 보입니다.

그리고 Presentation Layer 는 Domain Layer 에 대한 의존성을 가지고 있습니다.

그러나 Data 레이어에서 Domain 레이어의 작업을 하고싶은데 LiveData 객체를 작업하고 싶을 수 있지만 LiveData 는 비동기 데이터 스트림을 처리하도록 설계되지 않았습니다. LiveData transfromation 과 MediatorLiveData 등을 통해 이를 처리하게 할 수는 있겠지만, 모든 LiveData 의 관찰은 오직 Main Thread 에서만 진행되기 때문에 한계점을 갖고 있습니다.

안드로이드의 교과서인 디벨로퍼 사이트에서도 이에 대해 명시하고 있으며, Repository 에서는 LiveData 를 사용하지 않도록 권장하면서 동시에 [**Kotlin Flow**](https://github.com/tnvnfdla1214/ToDoApp) 를 사용하도록 권장하고 있습니다.

또한 Domain레이어는 안드로이드에 의존성을 가지지 않은 순수 Java 및 Kotlin 코드로만 구성합니다. 그런데 LiveData는 사용할 수 없습니다.

이에 따라 문제점을 요약하면 아래와 같습니다.
+ LiveData 는 UI 에 밀접하게 연관되어 있기 때문에 Data Layer 에서 비동기 방식으로 데이터를 처리하기에 자연스러운 방법이 없다.
+ LiveData 는 안드로이드 플랫폼에 속해 있기 때문에 순수 Java / Kotlin 을 사용해야 하는 Domain Layer 에서 사용하기에 적합하지 않다.

안드로이드 개발 언어로 코틀린이 자리 잡기 전까지는 위와 같은 문제점을 안고 있으면서도 안드로이드 진영에서는 별다른 선택권이 없었습니다. 그러나 코틀린 코루틴이 발전하면서, Flow 가 등장하게 되었고 많은 안드로이드 커뮤니티에서는 이 Flow 를 이용해서 LiveData 를 대체할 수 있을지에 대한 기대가 생기기 시작했습니다.

그러나 Flow 를 통해 LiveData 를 대체하는 것은 쉬운 일이 아니었습니다. 이유는 다음과 같습니다.
+ Flow 는 스스로 안드로이드 생명주기에 대해 알지 못합니다. 그래서 라이프사이클에 따른 중지나 재개가 어렵습니다.
+ Flow 는 상태가 없어 값이 할당된 것인지, 현재 값은 무엇인지 알기가 어렵습니다.
+ Flow 는 Cold Stream 방식(연속해서 계속 들어오는 데이터를 처리할 수 없음)으로, 연속해서 계속 들어오는 데이터를 처리할 수 없으며 collect 되었을 때만 생성되고 값을 반환합니다. 만약, 하나의 flow builder 에 대해 다수의 collector 가 있다면 collector 하나마다 하나씩 데이터를 호출하기 때문에 업스트림 로직이 비싼 비용을 요구하는 DB 접근이나 서버 통신 등이라면 여러 번 리소스 요청을 하게 될 수 있습니다.

이를 위해 kotlin 1.41 버전에 Stable API 로 등장한 것이 바로 **StateFlow** 와 **SharedFlow** 입니다.

## StateFlow
StateFlow 는 현재 상태와 새로운 상태 업데이트를 collector 에 내보내는 Observable 한 State holder flow 입니다. 그리고 LiveData 와 마찬가지로 value 프로퍼티를 통해서 현재 상태 값을 읽을 수 있습니다.

StateFlow 는 SharedFlow 의 한 종류이며, LiveData 에 가장 가깝습니다.

특징으로는 다음과 같습니다.

+ StateFlow 는 항상 값을 가지고 있고, 오직 한 가지 값을 가집니다.
+ StateFlow 는 여러 개의 collector 를 지원합니다. 이는 flow 가 공유된다는 의미이며 앞서 설명했던 flow 의 단점(3)과는 다르게 업스트림이 collector 마다 중복으로 처리되지 않습니다.
+ StateFlow 는 collector 수에 관계없이 항상 구독하고 있는 것의 최신 값을 받습니다.
+ StateFlow 는 flow builder 를 사용하여 빌드된 flow 가 cold stream 이었던 것과 달리, hot stream 입니다. 따라서 collector 에서 수집하더라도 생산자 코드가 트리거 되지 않고, 일반 flow 는 마지막 값의 개념이 없었던 것과 달리 StateFlow 는 마지막 값의 개념이 있으며 생성하자마자 활성화됩니다.
 
StateFlow 와 LiveData 는 둘 다 관찰 가능한 데이터 홀더 클래스이며, 앱 아키텍쳐에서 사용할 때 비슷한 패턴을 따릅니다. 즉, MVVM 에서 LiveData 사용되는 자리에 StateFlow 로 대체할 수 있습니다. 그리고 Android Studio Arctic fox 버전부터는 AAC Databinding 에도 StateFlow 가 호환됩니다.

그러나 StateFlow 와 LiveData 는 다음과 같이 다르게 작동합니다.

+ StateFlow 의 경우 초기 상태를 생성자에 전달해야 하지만, LiveData 의 경우는 전달하지 않아도 됩니다.
+ View 가 STOPPED 상태가 되면 LiveData.observe() 는 Observer 를 자동으로 등록 취소하는 반면, StateFlow 는 자동으로 collect 를 중지하지 않습니다. 만약 동일한 동작을 실행하려면 Lifecycle.repeatOnLifecycle 블록에서 흐름을 수집해야 합니다.

다음은 LiveData를 사용한 Domain 레이어와 UI 레이어 예시입니다.
## LiveData를 사용한 예시
<div align="center">
<img src = "https://user-images.githubusercontent.com/48902047/150665795-e91a7924-a84c-4ba3-8aeb-d7d971cf51b4.png">
</div>

데이터를 받아온 다음 뷰에 그릴 때 까지, 모든 부분에서 LiveData를 사용하는 예시를 한 번 보겠습니다. 여기에서 Data source 는 GeoQuery 라이브러리를 이용해 Firebase에서 데이터를 받아오는 부분입니다. 우리가 onGeoQueryReady() 또는 onGeoQueryError() 콜백에서 결과 값을 받으면, 우리는 그 결과 값을 모두 LiveData에 업데이트 할 것입니다. onGeoQueryReady 이후 "해당위치에 진입", "해당위치에서 나감", "이동 중" 같은 연속적인 값이 계속 LiveData에 업데이트 될 것입니다.

```Kotlin
@Singleton
class NearbyUsersDataSource @Inject constructor() {
    // Ideally, those should be constructor-injected.
    //이상적으로는 생성자 주입이 되어야 합니다.
    val geoFire = GeoFire(FirebaseDatabase.getInstance().getReference("geofire"))
    val geoLocation = GeoLocation(0.0, 0.0)
    val radius = 100.0
    
    val geoQuery = geoFire.queryAtLocation(geoLocation, radius)
    
    // Listener for receiving GeoLocations
    //GeoLocations 수신 수신기
    val listener: GeoQueryEventListener = object : GeoQueryEventListener {
        val map = mutableMapOf<Key, GeoLocation>()
        override fun onKeyEntered(key: String, location: GeoLocation) {
            map[key] = location
        }
        override fun onKeyExited(key: String) {
            map.remove(key)
        }
        override fun onKeyMoved(key: String, location: GeoLocation) {
            map[key] = location
        }
        override fun onGeoQueryReady() {
            _locations.value = State.Ready(map.toMap())
        }
        override fun onGeoQueryError(e: DatabaseError) {
            _locations.value = State.Error(map.toMap(), e.toException())
        }
    }

    // Listen for changes only while observed
    // 관찰되는 동안에만 변경 내용 수신
    private val _locations = object : MutableLiveData<State>() {
        override fun onActive() {
            geoQuery.addGeoQueryEventListener(listener)
        }

        override fun onInactive() {
            geoQuery.removeGeoQueryEventListener(listener)
        }
    }

    // Expose read-only LiveData
    // 읽기 전용 LiveData 노출
    val locations: LiveData<State> by this::_locations
    
    sealed class State(open val value: Map<Key, GeoLocation>) {
        data class Ready(
            override val value: Map<Key, GeoLocation>
        ) : State(value)
        
        data class Error(
            override val value: Map<Key, GeoLocation>,
            val exception: Exception
        ) : State(value)
    }
}
```
Repository, ViewModel, Activity는 아래와 같이 최대한 간단하게 구현됩니다.
```Kotlin
@Singleton
class NearbyUsersRepository @Inject constructor(
    nearbyUsersDataSource: NearbyUsersDataSource
) {
    val locations get() = nearbyUsersDataSource.locations
}
```
```Kotlin
class NearbyUsersViewModel @ViewModelInject constructor(
    nearbyUsersRepository: NearbyUsersRepository
) : ViewModel() {

    val locations get() = nearbyUsersRepository.locations
}
```
```Kotlin
@AndroidEntryPoint
class NearbyUsersActivity : AppCompatActivity() {
    
    private val viewModel: NearbyUsersViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        viewModel.locations.observe(this) { state: State ->
            // Update views with the data.   
        }
    }
}
```
이런 접근 방식을 쓰더라도 동작은 잘 될 것입니다. 도메인 레이어를 만들기로 결정하기 전까지는 말이죠. 도메인 레이어는 Repository 인터페이스를 가지고 있고, 플랫폼과 무관합니다. 또한, 데이터 소스에서 작업 쓰레드를 이용하여 별도 작업을 해야할 때는 기존에 LiveData를 쓰는 관례적인 방법대로는 그리 쉽지 않은 것을 발견하게 될겁니다.

다음은 LiveData 대신 Flow를 사용하는 예시입니다.

## Flow를 데이터소스와 Repository에 사용하기
<div align="center">
<img src = "https://user-images.githubusercontent.com/48902047/150665928-bf10e954-3502-4de3-8e38-1e2e325d7ad9.png">
</div>

Data Source가 Flow를 이용하도록 바꿔 보겠습니다. 우리는  flow builder, callbackFlow{} 를 쓸 수 있는데, 이것은 콜백 결과를  콜드 Flow 하나로 변환할 수 있습니다. Flow가 collect 되면, flow builder로 전달되었던 코드 블록을 실행하며, GeoQuery리스너를 추가하고 awaitClose {} 를 실행하게 됩니다. 이 코드는 Flow가 닫히기 전까지는 계속 멈춰있게 (suspend) 됩니다. (멈춰있다는 것은, 아무도 collect를 하지 않았거나, 아무런 에러도 없어서 캔슬되지도 않았기 때문이라는 뜻입니다). Flow가 닫히게 되면, 등록되었던 리스너가 제거되고 flow는 사라집니다.

```Kotlin
@Singleton
class NearbyUsersDataSource @Inject constructor() {
    // Ideally, those should be constructor-injected.
    val geoFire = GeoFire(FirebaseDatabase.getInstance().getReference("geofire"))
    val geoLocation = GeoLocation(0.0, 0.0)
    val radius = 100.0
    
    val geoQuery = geoFire.queryAtLocation(geoLocation, radius)
    
    private fun GeoQuery.asFlow() = callbackFlow {
        val listener: GeoQueryEventListener = object : GeoQueryEventListener {
            val map = mutableMapOf<Key, GeoLocation>()
            override fun onKeyEntered(key: String, location: GeoLocation) {
                map[key] = location
            }
            override fun onKeyExited(key: String) {
                map.remove(key)
            }
            override fun onKeyMoved(key: String, location: GeoLocation) {
                map[key] = location
            }
            override fun onGeoQueryReady() {
                emit(State.Ready(map.toMap()))
            }
            override fun onGeoQueryError(e: DatabaseError) {
                emit(State.Error(map.toMap(), e.toException()))
            }
        }
        
        addGeoQueryEventListener(listener)
        
        awaitClose { removeGeoQueryEventListener(listener) }
    }

    val locations: Flow<State> = geoQuery.asFlow()
    
    sealed class State(open val value: Map<Key, GeoLocation>) {
        data class Ready(
            override val value: Map<Key, GeoLocation>
        ) : State(value)
        
        data class Error(
            override val value: Map<Key, GeoLocation>,
            val exception: Exception
        ) : State(value)
    }
}
```
Repository와 ViewModel는 전혀 변경되는 부분이 없지만, Activity는 이제 LiveData가 아닌 Flow를 받게되기 때문에 약간의 수정이 필요합니다. LiveData를 observe하는 대신, Flow를 collect할 것입니다.
```Kotlin
@AndroidEntryPoint
class NearbyUsersActivity : AppCompatActivity() {
    
    private val viewModel: NearbyUsersViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        lifecycleScope.launchWhenStarted {
            viewModel.locations.collect {
                // Update views with the data.   
            }
        }
    }
}
```
우리가 Flow를 collect하기 위해 launchWhenStarted{} 를 사용했기 때문에, 코루틴은 Activity가 onStart() 라이프사이클 상태에 도달했을 때 자동으로 실행될 것입니다. 그리고 다시 라이프사이클 상태가 onStop()에 도달하면 멈출 것입니다. 이것은 LiveData가 LifeCycle을 자동으로 처리해주는 것과 똑같은 동작입니다.

<div align="center">
<img src = "https://user-images.githubusercontent.com/48902047/150665978-f28e1837-6de1-4a72-bab8-7dd0e8bd5a8a.png">
</div>

## View Layer에서 Flow를 사용했을 때 생기는 문제점은 무엇인가요?

이 방식의  첫 번째 문제점은, 라이프사이클을 처리할 때 발생합니다. (LiveData는 그걸 자동으로 처리해 주었지만...) 우리는 LiveData와 유사한 동작을 launchWhenStarted{} 를 이용하여 위에서 구현했었습니다.

하지만 다른 문제가 있습니다. Flow는 선언적이고, Collect 될 때만 실행됩니다 (Materialized). 만약 우리가 여러 개의 colletor를 가지고 있다면, 새로운 Flow 하나가 실행될 때 마다 다른 모든 Collector가 동작할 것입니다. 그것도 완벽히 독립적으로, 각각 실행될 것입니다.

어떤 동작이 완료되는 것을 기다리는 작업이라면, 그리고 그 동작이 데이터베이스 결과 값이거나 네트워크에서 얻어오는 결과 값이라면, 이것은 굉장히 비효율적인 작업입니다. 게다가 이 결과는, 결과값에 대해 한 번만 에러체크를 한다면 (역주: 한 번의 요청에 동일한 결과 값이 여러 번 오게 되니) 굉장히 이상한 상태가 될 수 있습니다. 우리의 연습예제에서는 이와 같이 각 Collector마다 하나의 listener를 추가해야만 했습니다. 아마 큰 이슈가 아닐 수도 있겠지만, 아마 메모리와 CPU를 낭비하게 될 것 입니다.

그래서 등장한 ShareFlow입니다. 다음은 SharedFlow 사용 예시 입니다.
## SharedFlow 사용 예시

SharedFlow 는 Flow 종류 중 하나로서, 그 자신을 여러 개의 Collector에 공유하기 때문에 오직 하나의 Flow만을 실행 (materized) 하게 해줍니다. 아무리 많은 Collector가 있어도 말이죠. 만약 SharedFlow 한 개를 정의하고 데이터베이스 결과 값을 공유하게 한 뒤, 여러 개의 Collector를 달아 준다면 데이터베이스 접근은 오직 한 번만 일어날 것입니다. 그리고 그 결과는 모든 Collector 들에게 공유될 것입니다.

StateFlow도 같은 동작을 하기 위해 사용될 수 있습니다. 이것은 특수한 SharedFlow 로서, ".value" 를 가지고 있습니다. (현재 값 반환) 그리고 SharedFlow의 설정값을 정확하게 정의합니다 (제약사항).

Flow를 SharedFlow로 변환할 수 있는 오퍼레이터는 아래와 같습니다.

```Kotlin
fun <T> Flow<T>.shareIn(
    scope: CoroutineScope, 
    started: SharingStarted, 
    replay: Int = 0
): SharedFlow<T> (source)
```
+ scope : 공유가 시작되는 Coroutine Scope.
+ started : 공유가 시작 및 중지되는 시기를 제어하는 전략을 설정하는 파라미터.
  + Lazily : 첫 번째 subscriber 가 나타나면 시작하고, scope 가 취소되면 중지.
  + Eagerly : 즉시 시작되며, scope 가 취소되면 중지.
  + WhileSubscribed : collector 가 없을 때 upstream flow 를 취소. 앱이 백그라운드로 전환되면 취소하게 하는 등의 전략 가능.
+ initialValue : StateFlow 의 초기 값.

이 코드를 우리의 Data Source에 적용해 봅시다.

이 동작의 스코프는 실행중인 (Materializing) 모든 계산 Flow가 완료될 때 까지입니다. 우리의 Data Source가 @Singleton이기 때문에, 우리는 어플리케이션 프로세스의 라이프 사이클 범위를 사용할 수 있습니다. 즉 LifecycleCoroutineScope은 프로세스가 생성될 때 함께 생성되고, 프로세스가 종료될 때만 함께 종료됩니다.

started 파라메터를 위해, 우리는 SharingStarted.WhileSubscribed()를 사용할 수 있는데, 이것은 Collector 가 0개 에서 1개가 되는 순간에 우리의 Flow의 공유를 시작할 수 있게 해 주며, (materializing) Collector가 0개가 되는 순간 공유를 멈춥니다. 이 동작은 LiveData의 동작과 유사한데, 우리가 GeoQuery에 추가했었던, onActive() 콜백을 받았을 때 리스너를 추가하고, onInactive()일 때 리스너를 제거하는 방식과 비슷합니다. 우리는 이것을 좀 더 적극적으로 시작시킬 수도 있고 (즉시 생성된 후 절대로 제거되지 않도록) 게으르게(lazy) (첫 번째로 결과값을 받았을 때 (collected) 생성되고, 절대 제거되지 않게) 설정할 수도 있습니다. 하지만 우리는 다운스트림이 Collect 되지 않았을 때 (역주: 결과 값을 받는 collector가 없을 때) 업스트림 데이터베이스 Collection을 멈추도록 (역주: 데이터베이스 요청을 하지 않도록) 설정하고 싶습니다.

***용어 설명: LiveData의 결과 값을 받는 부분을 observer로 부르고, cold Flow의 결과 값을 받는 부분을 collector로 불렀습니다. 마찬가지로, SharedFlow의 결과 값을 받는 부분을 subscriber로 부르겠습니다.***

replay 파라메터에는 1 값을 넣겠습니다. 새롭게 추가된 subscriber는 등록되는 즉시 직전 1번째에 받았던 값을 받게 됩니다.

```Kotlin
@Singleton
class NearbyUsersDataSource @Inject constructor() {
    // Ideally, those should be constructor-injected.
    val geoFire = GeoFire(FirebaseDatabase.getInstance().getReference("geofire"))
    val geoLocation = GeoLocation(0.0, 0.0)
    val radius = 100.0
    
    val geoQuery = geoFire.queryAtLocation(geoLocation, radius)
    
    private fun GeoQuery.asFlow() = callbackFlow {
        val listener: GeoQueryEventListener = object : GeoQueryEventListener {
            val map = mutableMapOf<Key, GeoLocation>()
            override fun onKeyEntered(key: String, location: GeoLocation) {
                map[key] = location
            }
            override fun onKeyExited(key: String) {
                map.remove(key)
            }
            override fun onKeyMoved(key: String, location: GeoLocation) {
                map[key] = location
            }
            override fun onGeoQueryReady() {
                emit(State.Ready(map.toMap())
            }
            override fun onGeoQueryError(e: DatabaseError) {
                emit(State.Error(map.toMap(), e.toException())
            }
        }
        
        addGeoQueryEventListener(listener)
        
        awaitClose { removeGeoQueryEventListener(listener) }
    }.shareIn(
         ProcessLifecycleOwner.get().lifecycleScope,
         SharingStarted.WhileSubscribed(),
         1
    )

    val locations: Flow<State> = geoQuery.asFlow()
                     
    sealed class State(open val value: Map<Key, GeoLocation>) {
        data class Ready(
            override val value: Map<Key, GeoLocation>
        ) : State(value)
        
        data class Error(
            override val value: Map<Key, GeoLocation>,
            val exception: Exception
        ) : State(value)
    }
}
```
이 코드를 읽어보면 SharedFlow를 flow collector 그 자체로 생각하는데 도움이 될 것입니다. 여기에서 SharedFlow는 cold flow를 hot flow로 바꿔주며, 이 스트림을 듣고 있는 많은 colletor에게 값을 공유해주고 있습니다. 위에서 내려오는 (upstream) cold flow를 아래에서 듣고 있는 (downstream) 많은 collector에게 전달해 주는, 중간자 적인 역할을 합니다.

이제, 아마 우리의 Activity는 아무런 수정이 필요 없다고 생각할 수도 있습니다. 아닙니다! 해야 할 것이 있는데요, launchWhenStarted{} 로 시작된 코루틴 flow를 collect 할 때는, 코루틴은 onStop() 에서 멈춰져야 하고 onStart() 에서 재시작되어야 하지만, 현재는 계속 flow를 subscribe 하고 있습니다. MutableSharedFlow\<T>를 보시면, MutableSharedFlow\<T>.subscriptionCount는 멈춰져 있는 코루틴에 대해서는 값이 변경되지 않습니다. SharingStarted.WhileSubscribed() 의 진정한 힘을 이용하기 위해서, 우리는 정확하게 onStop() 에서 unsubscribe 해야 하고, onStart()에서 다시 subscribe 해야 합니다. 이것은 collect 하는 코루틴을 캔슬하고, 다시 생성해야 한다는 것을 의미합니다.

(참고로 이 이슈 와 이 이슈를 보시면 더 자세한 내용을 확인하실 수 있습니다.)

범용적인 사용을 위한 클래스를 만들어 봅시다.

```Kotlin
@PublishedApi
internal class ObserverImpl<T> (
    lifecycleOwner: LifecycleOwner,
    private val flow: Flow<T>,
    private val collector: suspend (T) -> Unit
) : DefaultLifecycleObserver {

    private var job: Job? = null

    override fun onStart(owner: LifecycleOwner) {
        job = owner.lifecycleScope.launch {
            flow.collect {
                collector(it)
            }
        }
    }

    override fun onStop(owner: LifecycleOwner) {
        job?.cancel()
        job = null
    }

    init {
        lifecycleOwner.lifecycle.addObserver(this)
    }
}

inline fun <reified T> Flow<T>.observe(
    lifecycleOwner: LifecycleOwner,
    noinline collector: suspend (T) -> Unit
) {
    ObserverImpl(lifecycleOwner, this, collector)
}

inline fun <reified T> Flow<T>.observeIn(
    lifecycleOwner: LifecycleOwner
) {
    ObserverImpl(lifecycleOwner, this, {})
}
```
이제, 우리가 방금 만든 .observeIn(LifeCycleOwner) 를 Activity에서 사용하도록 수정해 보겠습니다.

```Kotlin
@AndroidEntryPoint
class NearbyUsersActivity : AppCompatActivity() {
    
    private val viewModel: NearbyUsersViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        viewModel
            .locations
            .onEach { /* new locations received */ }
            .observeIn(this)
    }
}
```
observeIn(LifecycleOwner) 를 통해 생성된 collector 코루틴은 LifecycleOwner의 Lifecycle이 CREATED 상태가 되었을 때 detroy 되고, (onStop()가 호출된 직후) LifecycleOwner의 Lifecycle이 STARTED 상태가 되었을 때 (onStart() 가 호출된 이후) 다시 생성됩니다.
  
##### 일러두기
왜 CREATED 상태인가요? STOPPED 상태일 때 호출되어야 하는게 아닌가요? 이것은 직관적인 것과는 정반대가 아닌가 하는 생각이 들겠지만, 이건 정확하게 맞는 말입니다. Lifecycle.State 은 다음과 같은 상태밖에 없습니다: CREATED, DESTROYED, INITIALIZED, RESUMED, STARTED. 
여기에는  STOPPED 또는 PAUSED 상태가 없습니다. lifecycle 이 onPause() 상태가 되면, 새로운 상태로 변하는 대신 STARTED 상태로 되돌아갑니다. onStop()가 되면, lifeCycle 상태는 CREATED 상태로 되돌아갑니다.

<div align="center">
<img src = "https://user-images.githubusercontent.com/48902047/150666322-86850d8c-1a23-452e-8b9e-49fc95c8235e.png">
</div>

이제 우리는 한 번만 생성(materializes) 되고, 여러 subscriber에 데이터를 공유해 주는 Data Source를 가지게 되었습니다. subscriber가 없다면 데이터를 요청하지 않고, 하나 이상의 subscriber가 다시 추가되면 곧바로 다시 데이터 요청을 시작하게 됩니다. 안드로이드 플랫폼에 의존적이지 않고 메인 쓰레드에 묶여있지 않습니다. (Flow 변환은 다른 쓰레드에서 할 수 있는데, 간단하게 flowOn(Dispatchers.IO) 또는 .flowOn(Dispatchers.Default)같은  .flowOn() 오퍼레이터를 쓰면 됩니다)

## 의문점
#### 하지만 만약 현재 Flow를 collect 하지 않으면서도 상태에 접근해야 할 때가 있다면?

만약 우리가 정말 마치 LiveData에서와 같이 Flow의 상태에 접근해야 한다면, 우리는 StateFlow를 이용할 수 있는데, 제약사항이 가해진 SharedFlow 입니다.

Flow를 생성하기 위해서 shareIn() 오퍼레이터를 사용하는 대신, 우리는  stateIn() 오퍼레이터를 사용할 수 있습니다.

```Kotlin
fun <T> Flow<T>.stateIn(
    scope: CoroutineScope, 
    started: SharingStarted, 
    initialValue: T
): StateFlow<T> (source)
```
이 메소드 파라메터에서 볼 수 있는 것 처럼, sharedIn() 와 stateIn() 사이에는 두 가지 기본적인 차이점이 있습니다.

1. stateIn() 은 replay 값을 설정 할 수 없습니다. StateFlow는 SharedFlow 에서 replay 값이 1로 고정된 것입니다. 이 말은 새로운 subscriber는 즉시 현재 상태 값을 받게 된다는 의미입니다.
2. stateIn()은 초기 값이 필요합니다. 이 말은 만약 당신이 처음에 설정해 줄 값이 없다면, StateFlow\<T> 에서 T 에 들어갈 타입을 null 일 수 있는 타입으로 설정하거나, sealed class를 설정해서 "비어있는 초기 값"을 정의해 줘야 한다는 의미입니다.


```Kotlin
// MutableStateFlow(initialValue) is a shared flow with the following parameters:
val shared = MutableSharedFlow(
    replay = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
shared.tryEmit(initialValue) // emit the initial value
val state = shared.distinctUntilChanged() // get StateFlow-like behavior
```
SharedFlow 를 사용하기에 적합한 곳은 StateFlow 에서 추가 버퍼가 필요하거나, 여러 개의 최근 값을 반환해야 하거나, 초기값을 무시해야 하는 곳입니다.

하지만, SharedFlow를 선택할 때 명백하게 타협해야 할 부분이 하나 있는데, 그것은 바로 StateFlow\<T>.value를 잃게 된다는 것입니다.


#### 둘 중 어느 것을 선택해야 하나요? StateFlow 아니면 SharedFlow?

이 문제에 대한 답을 찾는 쉬운 방법은 다른 질문을 몇 가지 해보는 것입니다.

"내가 정말 언제든지 myFlow.value 이런 방식으로 flow의 현재 상태에 접근해야만 하는가? "

만약 답이 아니오 라면, SharedFlow를 고려해도 좋을 것 같습니다.

"내가 반복되는 값을 발신하고, 그것을 collect 하는 동작을 지원해야 하는가?"

만약 답이 예 라면, SharedFlow를 고려하시기 바랍니다.

"내가 최신 값 1개가 아닌, 그 이전의 값까지 더 많은 값을 subscriber 에게 제공할 수 있어야 하는가?"

만약 답이 예 라면, SharedFlow를 고려하시기 바랍니다.

우리가 살펴본 것 처럼, StateFlow를 모든 곳에 사용하는 것은 올바른 답이 아닙니다.

1. StateFlow는 반복되는 결과 값을 무시합니다. (conflates)그리고 무시하지 않도록 설정할 수도 없습니다. 가끔은 반복해서 들어오는 값을 무시하지 않아야 할 때가 있습니다. 예를 들면, 접속 시도 결과 값 같은 경우에는 매번 실패할 때 마다 재시도해야 할 수 있습니다. 

2. 또한, StateFlow는 초기값이 필요합니다. SharedFlow는 .value같은 것이 존재하지 않기 때문에, 초기 값을 넣어 초기화할 필요가 없습니다. collector는 단시 첫 번째 값이 올 때 까지 기다립니다. 값이 들어오기 전에는 그 누구도 그 값에 접근하려하지 않을 것입니다. 만약 당신이 StateFlow 를 쓰면서 초기값이 없다면, 당신은 StateFlow의 타입 T 를 nullable 하게, T? 와 같이 설정한 다음 초기 값으로 null을 넣어줘야 합니다. (아니면, sealed class 를 생성한 다음, 초기 값으로 값이 없는 상태를 지정해 줘야 합니다)

3. 또한, replay 값을 설정해 줘야 합니다. SharedFlow 는 최근 n개의 값을 새로운 subscriber에게 전달해 줄 수 있습니다. StateFlow는 고정된 replay 값 1을 가지고 있고, 이 말은 현재 상태만을 공유할 수 있다는 의미입니다.

이 둘은 모두 SharingStarted ( Eagerly, Lazily 또는 WhileSubscribed()) 설정을 지원합니다. 저는 일반적으로 SharingStarted.WhileSubscribed()를 사용하고, Activity 에서 onStart / onStop 될 때 모든 collecter를 파괴 / 재생성 하기 때문에, 사용자가 앱을 사용하지 않을 때 데이터소스가 데이터를 요청하지 않습니다. (이것은 LiveData에서 onActive / onInactive 일 때, 리스너를 제거/다시 추가하는 동작과 비슷합니다)

StateFlow가 SharedFlow 보다 더 많이 가지고 있는 제약사항은, 아마 당신의 사용 목적에 정확하게 일치하지는 않을지도 모릅니다. 아마 당신은 좀 더 많은 것을 설정하기를 원할 것이고, SharedFlow를 사용하는 것을 택할지도 모릅니다. 개인적으로는, 저는 거의 myFlow.value와 같이 직접 값에 접근하지 않고, SharedFlow의 유연성을 좋아하기 때문에 SharedFlow를 선택하는 경우가 많습니다.

공식 문서에서 StateFlow 와 SharedFlow 에 대해 더 찾아보세요.

#### StateFlow가 아닌, SharedFlow가 필요한 실제 예제

아래와 같은, 구글 결제 클라이언트 라이브러리가 있다고 가정합시다. 우리는 MutableSharedFlow billingClientStatus 를 가지고 있는데, 이것은 현재 빌링 서비스의 연결 상태를 저장하고 있습니다. 우리는 이 변수의 초기값을 SERVICE_DISCONNECTED으로 설정합니다. 우리는 billingClientStatus 값을 받고, 결과 값이 OK라면 startConnection()를 빌링 서비스에 호출합니다. 연결시도가 실패하면, 우리는 SERVICE_DISCONNECTED를 발신합니다.

이 예제에서, billingClientStatus 가 MutableSharedFlow 가 아니라 MutableStateFlow  였다면, 초기 값이 SERVICE_DISCONNECTED 이고 우리가 시도한 요청의 결과 값 또한 동일하게 SERVICE_DISCONNECTED 이기 때문에 (요청이 실패했기 때문에) 업데이트 되지 않고, 결과적으로는 재접속 요청을 하지 않게 될 것 입니다.

```Kotlin
@Singleton
class Biller @Inject constructor(
    @ApplicationContext private val context: Context,
) : PurchasesUpdatedListener, BillingClientStateListener {
    
    private var billingClient: BillingClient =
        BillingClient.newBuilder(context)
            .setListener(this)
            .enablePendingPurchases()
            .build()
        
    private val billingClientStatus = MutableSharedFlow<Int>(
        replay = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    
    override fun onBillingSetupFinished(result: BillingResult) {
        billingClientStatus.tryEmit(result.responseCode)
    }

    override fun onBillingServiceDisconnected() {
        billingClientStatus.tryEmit(BillingClient.BillingResponseCode.SERVICE_DISCONNECTED)
    }
    
    // ...
    
    // Suspend until billingClientStatus == BillingClient.BillingResponseCode.OK
    private suspend fun requireBillingClientSetup(): Boolean =
        withTimeoutOrNull(TIMEOUT_MILLIS) {
            billingClientStatus.first { it == BillingClient.BillingResponseCode.OK }
            true
        } ?: false
   
    init {
        billingClientStatus.tryEmit(BillingClient.BillingResponseCode.SERVICE_DISCONNECTED)
        billingClientStatus.observe(ProcessLifecycleOwner.get()) {
            when (it) {
                BillingClient.BillingResponseCode.OK -> with (billingClient) {
                    updateSkuPrices()
                    handlePurchases()
                }
                else -> {
                    delay(RETRY_MILLIS)
                    billingClient.startConnection(this@Biller)
                }
            }
        }
    }

    private companion object {
        private const val TIMEOUT_MILLIS = 2000L
        private const val RETRY_MILLIS = 3000L
    }
}
```

이러한 경우, 우리는 SharedFlow를 써야하고, 이것은 동일한 값이 반복해서 들어오더라도 무시하지 않고 처리합니다.

#### GeoFire 에서 사용했던 Use-Case

당신이 GeoFire 를 사용할 필요가 있다면, 제가 만든 라이브러리인 geofire-ktx 를 써보시기 바랍니다. 이 라이브러리를 이용하면 GeoQuery의 결과를 Flow로 받을 수 있습니다. 또한 DataSnapshot 요청을 다른 DatabaseReference 루트에 GeoFire 루트와 동일한 child key로 요청할 수 있습니다. 마치 이것이 GeoQuery의 일반적인 사용법인 것 처럼 말입니다. 그리고 결과 값을 DataSnapshot 타입이 아닌 다른 클래스로도 받을 수 있습니다. 이것은 Flow 변환을 통해 제공됩니다. 라이브러리의 source code 는 이 글에 있는 예제의 완전한 버전입니다.



