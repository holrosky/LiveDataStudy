# LiveData
**수명 주기를 인식하는 관찰 가능한 데이터 홀더 클래스**

+ ***수명 주기 인식***
  + LiveData는 자신을 관찰하고 있는 Observer 에게 변경을 알려주는데, 만약 Observer 가 비활성화 상태라면 변경을 알려주지 않는다. 즉, 활성화(***STARTED*** 혹은 ***RESUME*** 상태)된 Observer 에게만 변경사항을 전달한다. DESTROYED 된 observer 는 자동으로 해제되기 때문에 메모리 누수로 부터 안전하다. 이러한 이유로 LiveData를 관찰하기 위해서는 아래와 같은 코드가 사용되는데, observer() 메서드에 LifecycleOwner와 observer를 전달하는 것을 확인할 수 있다.
     ```kotlin
     val nameObserver = Observer<String> { newName ->
              // Update the UI, in this case, a TextView.
              nameTextView.text = newName
          }
  
     // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
     model.currentName.observe(this, nameObserver)
     ```
    그렇다면 Compose 에서는 어떻게 아래와 같이 LiveData를 LifecycleOwner 없이 observeAsState() 만으로 가능한걸까?
    ```kotlin
    val state = viewModel.state.observeAsState()
    ```
    observeAsState() 함수 내부 구조는 아래와 같다.
    ```kotlin
    @Composable
    fun <R, T : R> LiveData<T>.observeAsState(initial: R): State<R> {
        val lifecycleOwner = LocalLifecycleOwner.current
        val state = remember {
            @Suppress("UNCHECKED_CAST") /* Initialized values of a LiveData<T> must be a T */
            mutableStateOf(if (isInitialized) value as T else initial)
        }
        DisposableEffect(this, lifecycleOwner) {
            val observer = Observer<T> { state.value = it }
            observe(lifecycleOwner, observer)
            onDispose { removeObserver(observer) }
        }
        return state
    }
    ```
    Composable 함수는 LocalLifecycleOwner을 통해 자체적으로 수명주기를 인식할 수 있기때문에 LiveData를 관찰할 때 LifecycleOwner를 전달해주지 않아도 내부에서 스스로 관리를 한다. DisposableEffect 을 활용하여 Composable 처음 실행될 때 observer 를 등록하고 Composable 이 소멸될 때 해제해주고 있다.
    
+ ***Compose 에서 LiveData 가 필요할까?***
  + Compose 에는 State 객체가 존재하며 MutableLiveData 처럼 State 를 확장한 MutableStateOf 가 존재한다. LiveData 는 configuration change 에서도 데이터를 보존하고 State 또한 rememberSaveable 을 사용하면 동일한 효과를 얻을 수 있다. State 는 LiveData 처럼 생명주기를 인식하지는 않지만 StateFlow 를 사용하면 collectAsStateWithLifecycle() 함수를 통해 생명주기를 인식할 수 있다. 그렇다면, Compose 의 State 와 Flow 의 StateFlow 가 존재하는데 굳이 LiveData 를 사용해야 하는 경우가 과연 있을까? 
    
# MutableLiveData
**LiveData 를 확장한 변경 가능한 클래스**

+ ***변경 가능***
  + LiveData 는 MVVM 패턴 구현에서 자주 볼 수 있다. 보통 ViewModel 안에서 데이터 홀더 역할로 사용이되는데, ViewModel 은 Activity 혹은 Frament 과 같은 UI의 생명주기와 별도의 생명주기를 가지기 때문에 configruation change (ex. 화면회전) 에도 대처가 가능하며 View 들간의 데이터 공유도 가능하게 해준다.
    ```kotlin
    private val _contacts = MutableLiveData<List<Contact>>()
    val contacts: LiveData<List<Contact>> = _contacts
    ```
    보통 ViewModel 안에 위같이 사용이 된다.

    _***왜 저렇게 사용이 될까?***_

    LiveData 는 보통 View 가 관찰을 하고 있는 데이터이다. 즉, 외부에 노출이 된 데이터인데, 외부에서는 변경을 못하게 하기위해 불변 타입인 LiveData를 사용한다. 해당 값을 변경하기 위해 ViewModel 내부에서 변경이 가능한 MutableLiveData를 두고 해당 클래스가 지원하는 setValue() 혹은 postValue()로 값을 변경한다. 이렇게 함으로써 외부에서의 변경을 막을 수 있다.
+ ***setValue() || postValue()***
  + 두 함수 모두 LiveData의 값을 변경하기 위해 사용된다. 하지만 setValue()는 메인 쓰레드에서, postValue()는 백그라운드 쓰레드에서 작동한다는 점에서 차이가 있다. 아래는 postValue() 함수와 setValue() 함수의 내부구조이다.
   ```kotlin
       /**
     * Posts a task to a main thread to set the given value. So if you have a following code
     * executed in the main thread:
     * <pre class="prettyprint">
     * liveData.postValue("a");
     * liveData.setValue("b");
     * </pre>
     * The value "b" would be set at first and later the main thread would override it with
     * the value "a".
     * <p>
     * If you called this method multiple times before a main thread executed a posted task, only
     * the last value would be dispatched.
     *
     * @param value The new value
     */
    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }

    /**
     * Sets the value. If there are active observers, the value will be dispatched to them.
     * <p>
     * This method must be called from the main thread. If you need set a value from a background
     * thread, you can use {@link #postValue(Object)}
     *
     * @param value The new value
     */
    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }
   ```
   주석에서도 알 수 있듯 postValue 는 작업을 메인 쓰레드에 작업을 "예약" 하는 느낌이다. 메인 쓰레드를 사용 가능할 때 까지 기다렸다가 메인 쓰레드가 사용 가능한 상태가 되면 작업을 요청한다. 만약 postValue() 함수가 여러번 호출되고 많은 작업이 예약된다면 가장 최근 작업만 메인 쓰레드가 처리하게 한다. 반면 setValue() 는 메인 쓰레드에서만 호출되어야 하며, 그렇지 않으면 런타임에러가 발생한다. 비동기 작업이 많아지고 그에 따라 UI 변경을 하려는 쓰레드가 많아진다면 UI 변경 결과가 예상하기 힘들 수 있으므로 postValue() 사용은 가급적이면 지양하는게 좋지않을까? 


