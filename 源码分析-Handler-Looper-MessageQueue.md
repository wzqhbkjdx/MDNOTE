### Looper

参考该博客：http://www.cnblogs.com/codingmyworld/archive/2011/09/12/2174255.html

在线程中有一个Looper对象，它内部维护了一个消息队列，且一个线程只能有一个Looper对象

因为Looper的prepare()方法的核心就是将looper对象定义为ThreadLocal

使用Looper类创建Looper线程很简单：

	public class LooperThread extends Thread {
    		@Override
    		public void run() {
       	 // 将当前线程初始化为Looper线程
        	Looper.prepare();
        
        	// ...其他处理，如实例化handler
        
        	// 开始循环处理消息队列
       	 Looper.loop();
    		}
     }
	
	private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    关于ThreadLocal
     Implements a thread-local storage, that is, a variable for which each thread has 
     its own value. All threads share the same {@code ThreadLocal} object, but each 
     sees a different value when accessing it, and changes made by one thread do 
     not affect the other threads.
     
Loop()方法，调用loop方法后，Looper线程就开始工作了，不断从自己的MessageQueue中取出队头的消息执行

	/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }

关于Loopper的总结

* 每个线程有且最多只能有一个Looper对象，它是一个ThreadLocal
* Looper内部有一个消息队列，loop()方法调用后线程开始不断从队列中取出消息执行 msg.target.dispatchMessage(msg); 交由Handler来执行

###Handler

Handler向MessageQueue添加消息，处理消息，只处理自己发出的消息，Handler创建的时候会关联一个Looper，默认的构造方法会关联当前线程的Looper。

	A Handler allows you to send and process {@link Message} and Runnableobjects associated 
	with a thread's {@link MessageQueue}.  Each Handler instance is associated with a single thread 
	and that thread's message queue.  When you create a new Handler, it is bound to the thread / message queue of the thread that is creating it -- 
	from that point on, it will deliver messages and runnables to that message queue and execute them as they come out of the message queue. There are two main uses for a Handler: (1) to schedule messages and runnables to be executed as some point in the future; and (2) to enqueue an action to be performed on a different thread than your own.
	
	public class Handler {
		
		final MessageQueue mQueue; //关联的MessageQueue
    		final Looper mLooper; //关联的Looper
    		final Callback mCallback; //回调函数
    		
    		public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
		// 默认将关联当前线程的looper
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        // 重要！！！直接把关联looper的MQ作为自己的MQ，因此它的消息将发送到关联looper的MQ上
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
		
	}
	
下面我们就可以为之前的类加入Handler

	public class LooperThread extends Thread {
    private Handler handler1;
    private Handler handler2;

    @Override
    public void run() {
        // 将当前线程初始化为Looper线程
        Looper.prepare();
        
        // 实例化两个handler
        handler1 = new Handler();
        handler2 = new Handler();
        
        // 开始循环处理消息队列
        Looper.loop();
    }
    }
    //一个线程可以有多个Handler，但是只能有一个Looper！
   
有了handler之后，我们就可以使用 post(Runnable), postAtTime(Runnable, long), postDelayed(Runnable, long), sendEmptyMessage(int), sendMessage(Message), sendMessageAtTime(Message, long)和 sendMessageDelayed(Message, long)这些方法向MQ上发送消息了。光看这些API你可能会觉得handler能发两种消息，一种是Runnable对象，一种是message对象，这是直观的理解，但其实post发出的Runnable对象最后都被封装成message对象了

Handler处理消息

消息的处理是通过核心方法dispatchMessage(Message msg)与钩子方法handleMessage(Message msg)完成的

	// 处理消息，该方法由looper调用
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            // 如果message设置了callback，即runnable消息，处理callback！
            handleCallback(msg);
        } else {
            // 如果handler本身设置了callback，则执行callback
            if (mCallback != null) {
                 /* 这种方法允许让activity等来实现Handler.Callback接口，避免了自己编写handler重写handleMessage方法。见http://alex-yang-xiansoftware-com.iteye.com/blog/850865 */
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            // 如果message没有callback，则调用handler的钩子方法handleMessage
            handleMessage(msg);
        }
    }
    
    // 处理runnable消息
    private final void handleCallback(Message message) {
        message.callback.run();  //直接调用run方法！
    }
    // 由子类实现的钩子方法
    public void handleMessage(Message msg) {
    }
    
Handler可以在任意线程发送消息，这些消息会被添加到关联的MessageQueue上

Handler是在它关联的looper线程中处理消息的
	

	
	
