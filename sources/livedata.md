### LiveData使用介绍

- LiveData 可以观察数据变化的框架
- LiveData 内部维护一个Observers列表

     private SafeIterableMap<Observer<T>, LifecycleBoundObserver> mObservers =
                new SafeIterableMap<>();

- 如果一个Observer的生命周期处于STARTED或RESUMED状态，那么LiveData将认为这个Observer处于活跃状态.LiveData仅通知活跃的Observer去更新UI。非活跃状态的Observer，即使订阅了LiveData，也不会收到更新的通知。

- observe 观察数据会注入一个LifecycleOwner，和一个observer,同时使用装饰者模式，将二者封装到一个新的Observer，添加到Observers列表中
  然后使用lifecycle 添加这个wrapper用来监测生命周期，当有新的数据的时候，会遍历列表，先判断owner的状态是否可用，然后进行分发

    
        public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
            if (owner.getLifecycle().getCurrentState() == DESTROYED) {
                // ignore
                return;
            }
            LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
            LifecycleBoundObserver existing = mObservers.putIfAbsent(observer, wrapper);
            if (existing != null && existing.owner != wrapper.owner) {
                throw new IllegalArgumentException("Cannot add the same observer"
                        + " with different lifecycles");
            }
            if (existing != null) {
                return;
            }
            owner.getLifecycle().addObserver(wrapper);
        }

- LiveData内部没有任何线程安全控制，那么线程安全吗？ 答案当然是安全的
        如果event 发时所在的线程是主线程，并且是onStart OnResume onPause状态时，数据就会被执行，而不会考虑再次过程中关闭页面，导致onStop,onDestroy,因为，我们知道UI线程是一个一个回调的，比如想想一下onResume发出一个耗时事件，0.01秒，点击返回键使之结束;实际情况也会先执行耗时任务，执行完后再执行结束函数，最后再回调onStop onDestory等方法

#### LiveData的优点
在项目中使用LiveData，会有以下优点：

确保UI符合数据状态
LiveData遵循观察者模式。当生命周期状态改变时，LiveData会向Observer发出通知。您可以把更新UI的代码合并在这些Observer对象中。不必去考虑导致数据变化的各个时机， -次数据有变化，观察者都会去更新UI。
没有内存泄漏
观察会绑定具有生命周期的对象，并在这个绑定的对象被销毁后自行清理。
不会因停止活动发生而崩溃
如果观察的生命周期处于非活跃状态，例如在后退堆栈中的活动，就不会收到任何LiveData事件的通知。
不需要手动处理生命周期
UI组件只需要去观察相关数据，不需要手动去停止或恢复观察.LiveData会进行自动管理这些事情，因为在观察时，它会感知到相应组件的生命周期变化。
始终保持最新的数据
如果一个对象的生命周期变到非活跃状态，它将在再次变为活跃状态时接收最新的数据。例如，后台活动在返回到前台后立即收到最新数据。
应对正确配置更改
如果一个活动或片段由于配置更改（如设备旋转）而重新创建，它会立即收到最新的可用数据。
共享资源
您可以使用单例模式扩展LiveData对象并包装成系统服务，以便在应用程序中进行共享.LiveData对象一旦连接到系统服务，任何需要该资源的Observer都只需观察这个LiveData对象。有关更多信息，请参阅扩展LiveData。
使用LiveData对象

#### 按照以下步骤使用LiveData对象：

创建一个LiveData的实例来保存特定类型的数据。这通常在ViewModel类中完成。
创建一个定义了onChanged（）方法的Observer对象，当LiveData对象保存的数据发生变化时，onChanged（）方法可以进行相应的处理。您通常在UI控制器（如Activity或Fragment）中创建Observer对象。
使用observe（）方法将Observer对象注册到LiveData对象。observe（）方法还需要一个LifecycleOwner对象作为参数.Observer对象订阅了LiveData对象，便会在数据发生变化时发出通知。您通常需要UI控制器（如活动或片段）中注册观测对象。
注意：您可以使用observeForever（Observer）方法注册一个没有关联LifecycleOwner对象的Observer。在这种情况下，Observer被认为始终处于活动状态，因此当有数据变化时总是会被通知。您可以调用removeObserver（观察员）方法移除这些观察员。

  当你更新LiveData对象中存储的数据时，所有注册了的Observer，只要所绑定的LifecycleOwner处于活动状态，就会被触发通知
  .LiveData允许UI控制器Observer订阅更新。当LiveData对象所保存的数据发生变化时，UI会在响应中自动更新
