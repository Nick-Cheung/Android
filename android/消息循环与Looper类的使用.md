## 消息循环与Looper类的应用

### - 了解Looper

　　Looper是用于给一个线程添加一个消息队列(MessageQueue)，并且循环等待，当有消息时会唤起线程来处理消息的一个工具，直到线程结束为止。通常情况下不会用到Looper，因为对于Activity，Service等系统组件，Frameworks已经为我们初始化好了线程(俗称的UI线程或主线程)，在其内含有一个Looper，和由Looper创建的消息队列，所以主线程会一直运行，处理用户事件，直到某些事件(BACK）退出。

　　如果，我们需要新建一个线程，并且这个线程要能够循环处理其他线程发来的消息事件，或者需要长期与其他线程进行复杂的交互，这时就需要用到Looper来给线程建立消息队列。

　　使用Looper也非常的简单，它的方法比较少，最主要的有四个:

　　public static prepare();

　　public static myLooper();

　　public static loop();

　　public void quit();

使用方法如下：

1. 在每个线程的run()方法中的最开始调用Looper.prepare()，这是为线程初始化消息队列。

2. 之后调用Looper.myLooper()获取此Looper对象的引用。这不是必须的，但是如果你需要保存Looper对象的话，一定要在prepare()之后，否则调用在此对象上的方法不一定有效果，如looper.quit()就不会退出。

3. 在run()方法中添加Handler来处理消息

4. 添加Looper.loop()调用，这是让线程的消息队列开始运行，可以接收消息了。

5. 在想要退出消息循环时，调用Looper.quit()注意，这个方法是要在对象上面调用，很明显，用对象的意思就是要退出具体哪个Looper。如果run()中无其他操作，线程也将终止运行。


### - Looper实例
```java
public class LooperDemoActivity extends Activity {  
    private WorkerThread mWorkerThread;  
    private TextView mStatusLine;  
    private Handler mMainHandler;  
      
    @Override  
    public void onCreate(Bundle icicle) {  
	    super.onCreate(icicle);  
	    setContentView(R.layout.looper_demo_activity);  
	    mMainHandler = new Handler() {  
	        @Override  
	        public void handleMessage(Message msg) {  
	        String text = (String) msg.obj;  
	        if (TextUtils.isEmpty(text)) {  
	            return;  
	        }  
	        mStatusLine.setText(text);  
	        }  
	    };  
	      
	    mWorkerThread = new WorkerThread();  
	    final Button action = (Button) findViewById(R.id.looper_demo_action);  
	    action.setOnClickListener(new View.OnClickListener() {  
	        public void onClick(View v) {  
	        mWorkerThread.executeTask("please do me a favor");  
	        }  
	    });  
	    final Button end = (Button) findViewById(R.id.looper_demo_quit);  
	    end.setOnClickListener(new View.OnClickListener() {  
	        public void onClick(View v) {  
	        	mWorkerThread.exit();  
	        }  
	    });  
	    mStatusLine = (TextView) findViewById(R.id.looper_demo_displayer);  
	    mStatusLine.setText("Press 'do me a favor' to execute a task, press 'end of service' to stop looper thread");  
    }  
      
    @Override  
    public void onDestroy() {  
	    super.onDestroy();  
	    mWorkerThread.exit();  
	    mWorkerThread = null;  
    }  
      
    private class WorkerThread extends Thread {  
	    protected static final String TAG = "WorkerThread";  
	    private Handler mHandler;  
	    private Looper mLooper;  
	      
	    public WorkerThread() {  
	        start();  
	    }  
	      
	    public void run() {  
	        
	        Looper.prepare();  
	        
	        mLooper = Looper.myLooper();  
	        mHandler = new Handler(mLooper) {  
	            @Override  
	            public void handleMessage(Message msg) {  
	                StringBuilder sb = new StringBuilder();  
	                sb.append("it is my please to serve you, please be patient to wait!\n");  
	                Log.e(TAG, "workerthread, it is my please to serve you, please be patient to wait!");  
	
	                for (int i = 1; i < 100; i++) {  
	                    sb.append(".");  
	                    Message newMsg = Message.obtain();  
	                    newMsg.obj = sb.toString();  
	                    mMainHandler.sendMessage(newMsg);  
	                	Log.e(TAG, "workthread, working" + sb.toString());  
	                	SystemClock.sleep(100);  
	            	}  
	
	            	Log.e(TAG, "workerthread, your work is done.");  
	            	sb.append("\nyour work is done");  
	            	Message newMsg = Message.obtain();  
	            	newMsg.obj = sb.toString();  
	            	mMainHandler.sendMessage(newMsg);  
	        	}  
	        };  
	        Looper.loop();  
	    }  
	      
	    public void exit() {  
	        if (mLooper != null) {  
	        mLooper.quit();  
	        mLooper = null;  
	        }  
	    }  
	      
	   
	    public void executeTask(String text) {  
	        if (mLooper == null || mHandler == null) {  
	            Message msg = Message.obtain();  
	            msg.obj = "Sorry man, it is out of service";  
	            mMainHandler.sendMessage(msg);  
	            return;  
	        }  
	        Message msg = Message.obtain();  
	        msg.obj = text;  
	        mHandler.sendMessage(msg);  
	    }  
	}  
} 
```

　　这个实例中，主线程中执行任务仅是给服务线程发一个消息同时把相关数据传过去，数据会打包成消息对象(Message)，然后放到服务线程的消息队列中，主线程的调用返回，此过程很快，所以不会阻塞主线程。服务线程每当有消息进入消息队列后就会被唤醒从队列中取出消息，然后执行任务。服务线程可以接收任意数量的任务，也即主线程可以不停的发送消息给服务线程，这些消息都会被放进消息队列中，服务线程会一个接着一个的执行它们----直到所有的任务都完成(消息队列为空，已无其他消息)，服务线程会再次进入休眠状态----直到有新的消息到来。

　　如果想要终止服务线程，在mLooper对象上调用quit()，就会退出消息循环，因为线程无其他操作，所以整个线程也会终止。

　　需要注意的是当一个线程的消息循环已经退出后，不能再给其发送消息，否则会有异常抛出"RuntimeException: Handler{4051e4a0} sending message to a Handler on a dead thread"。所以，建议在Looper.prepare()后，调用Looper.myLooper()来获取对此Looper的引用，一来是用于终止(quit()必须在对象上面调用）; 另外就是用于接收消息时检查消息循环是否已经退出(如上例)。 