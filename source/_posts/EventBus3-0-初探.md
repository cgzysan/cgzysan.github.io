---
title: EventBus3.0 初探
date: 2018-03-30 15:20:21
tags: Android 组件间通信
---

### 前言

> 最近项目中，有好些后台 Service，就免不了 Activity 和 Service 的通信，也就有了不少 Broadcast，写了不少重复代码，就产生了使用 EventBus 的意思，而且现在更新到了 EventBus 3.0，用法有了些许不同，不过相比之前更方便。

[EventBus Github](https://github.com/greenrobot/EventBus)

### EventBus 有什么用

> 在项目开发中，肯定有不少各组件之间、组件与后台线程之间的通信，比如说子线程请求数据，然后通过 Handler 在主线程（也就是 UI 线程）更新UI，当然这种情况如果可以我会使用 Retrofit + Rxjava 实现，主要是 Fragment 和 Activity，Fragment 和 Fragment 之间的通信，特别是Fragment 嵌套比较深的布局，如果通过 Listener 回调传递事件，需要一层一层的往上传递，会造成代码的高度耦合，同时代码量也会大大增加。

#### EventBus 的优点

- EventBus 能够简化各组件间的通信，简化了代码
- 能够有效的分离事件发送方和接收方，解耦了代码
- 速度快，轻量

### 四步骤

#### 1. 定义事件

```java
public static class MessageEvent { /* Additional fields if needed */ }
```

#### 2. 创建 subscribers，定义并注释订阅方法，同时指定一个 Thread mode

> 在 EventBus3.0 之前必须要定义以 onEvent 开头，特定的那几个方法：onEvent，onEventMainThread，onEventBackgroundThread，onEventAsync，但是在 3.0 之后事件处理的方法名可以随便取，不过需要加上注解 @subscribe() ，并指定线程类型，默认是 POSTING。

```java
@Subscribe(threadMode = ThreadMode.MAIN)  
public void onMessageEvent(MessageEvent event) {/* Do something */};
```

#### 3. 注册和注销 subscriber，Fragment 和 Activity 应该根据生命周期注册

```java
 @Override
 public void onStart() {
     super.onStart();
     EventBus.getDefault().register(this);
 }

 @Override
 public void onStop() {
     super.onStop();
     EventBus.getDefault().unregister(this);
 }
```

#### 4. 发布事件

```java
EventBus.getDefault().post(new MessageEvent());
```

#### 线程模型 ThreadMode

- ThreadMode.POSTING（默认）  事件处理函数的线程和发布事件的线程在同一线程
- ThreadMode.MAIN  事件处理函数在主线程（UI线程），不能进行耗时操作
- ThreadMode.MAIN_ORDERED  事件处理函数在顺序处理的主线程，事件处理程序必须迅速返回，以避免阻塞主线程
- ThreadMode.BACKGROUND  事件处理函数在后台线程，可以进行耗时操作，但不能进行UI操作
- ThreadMode.ASYNC  无论事件发布的线程是哪一个，事件处理函数始终都会新建一个子线程运行，不能进行UI操作

### 实例

实现一个简单实例，在 Activity 中启动一个服务，在服务中，假设过去了 5s，返回给 Activity 一个消息，让 Activity 在页面中显示出来

#### 不使用EventBus，通过广播实现

##### Activity

```java
public class BroadcastActivity extends AppCompatActivity {

    private TextView mTvMsg;
    private BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            switch (action) {
                case "TIMEOUT":
                    Log.i("ysan", "Current thread = " + Thread.currentThread().getName());
                    String hello = intent.getStringExtra("hello");
                    mTvMsg.setText(hello);
                    break;
            }
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_message);

        mTvMsg = findViewById(R.id.tv_message);
        Intent timeService = new Intent(this, BroadService.class);
        startService(timeService);
    }

    @Override
    protected void onResume() {
        super.onResume();
        IntentFilter filter = new IntentFilter("TIMEOUT");
        registerReceiver(mReceiver, filter);
    }

    @Override
    protected void onPause() {
        super.onPause();
        unregisterReceiver(mReceiver);
    }
}
```

##### Service

```java
public class BroadService extends Service {

    @Override
    public void onCreate() {
        super.onCreate();
        new Thread(){
            @Override
            public void run() {
                try {
                    Thread.sleep(5000);
                    Intent timeout = new Intent("TIMEOUT");
                    timeout.putExtra("hello", "hello Broadcast");
                    sendBroadcast(timeout);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return super.onStartCommand(intent, flags, startId);
    }
}
```

#### 通过EventBus实现

##### Message类

传递数据的 bean 类，包装了用来传输的数据

```java
public class Message {
    private String msg;

    public Message(String msg) {
        this.msg = msg;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

##### Activity

```java
public class EventBusActivity extends AppCompatActivity {

    private TextView mTvMsg;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_message);

        mTvMsg = findViewById(R.id.tv_message);
        Intent timeService = new Intent(this, EventbusService.class);
        startService(timeService);
        Intent newsService = new Intent(this, NewsService.class);
        startService(newsService);
    }

    @Override
    protected void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (EventBus.getDefault().isRegistered(this))
            EventBus.getDefault().unregister(this);
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(Message msg) {
        String s = msg.getMsg();
        Log.i("ysan", "线程 >>>" + Thread.currentThread().getName() + "，收到消息" + s);
        mTvMsg.setText(s);
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onNewsEvent(News news) {
        String s = news.getInfo();
        Log.i("ysan", "线程 >>>" + Thread.currentThread().getName() + "，收到消息" + s);
        mTvMsg.setText(s);
    }

    @Subscribe(threadMode = ThreadMode.POSTING)
    public void onNewsevent(News news) {
        String s = news.getInfo();
        Log.i("ysan", "线程 >>>" + Thread.currentThread().getName() + "，收到消息" + s);
    }
}
```

##### Service

```java
public class EventbusService extends Service {

    @Override
    public void onCreate() {
        super.onCreate();
        new Thread(){
            @Override
            public void run() {
                try {
                    Thread.sleep(5000);
                    Message msg = new Message("hello EventBus");
                    EventBus.getDefault().post(msg);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return super.onStartCommand(intent, flags, startId);
    }
}
```

[完整代码 Github地址](https://github.com/cgzysan/EventBusDemo)