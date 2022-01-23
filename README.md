# StateFlow

이포스터는 LiveData에서 StateFlow로 바꿔야하는 이유와 바꾸는 방법에 대해서 작성합니다.

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
+ Flow 는 Cold Stream 방식으로, 연속해서 계속 들어오는 데이터를 처리할 수 없으며 collect 되었을 때만 생성되고 값을 반환합니다. 만약, 하나의 flow builder 에 대해 다수의 collector 가 있다면 collector 하나마다 하나씩 데이터를 호출하기 때문에 업스트림 로직이 비싼 비용을 요구하는 DB 접근이나 서버 통신 등이라면 여러 번 리소스 요청을 하게 될 수 있습니다.

이를 위해 kotlin 1.41 버전에 Stable API 로 등장한 것이 바로 **StateFlow** 와 **SharedFlow** 입니다.

## StateFlow
