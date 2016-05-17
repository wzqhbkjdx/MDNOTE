###Android系统源代码情景分析

>Activity 组件的启动过程

MainActivity的android:process 属性设置为 "mainprocess", Activity是通过Launcher组件来启动的，Launcher组件通过 Activity 管理服务ActivityManagerService来启动Activity ，ActivityManagerService在启动MainActivity组件时，就会发现系统中并不存在一个“mainprocess”进程，这是它就会先创建这个应用程序的进程，然后再讲MainActivity的组件启动起来；

由于MainActivity、Launcher、ActivityManagerService三者分别运行在不同的进程中，因此MainActivity的启动过程就涉及到了三个进程，这三个进程是通过Binder进程间通信机制来完成MainActivity组件的启动过程的。

Launcher组件启动MainActivity组件的过程如下：

（1）Launcher组件向ActivityManagerService发送一个启动MainActivity组件的进程间通信请求；

（2）ActivityManagerService首先将要启动的MainActivity的组件信息保存下来，然后再向launcher组件发送一个进入终止状态的进程间通信请求；

（3）Launcher组件进入终止状态后，就会向ActivityManagerService发送一个已进入终止状态的进程间通信请求，以便于ActivityManagerService继续执行启动MainActivity组件的操作；

（4）ActivityManagerService发现用来运行MainActivity组件的应用程序进程不存在，因此，它就会先启动一个新的应用程序进程；

（5）新的应用程序进程启动完成后，就会向ActivityManagerService发送一个启动完成的进程间通信，以便于ActivityManagerService继续执行启动MainActivity组件的操作；

（6）ActivityManagerService将保存下来的MainActivity的组件信息发送给刚刚为MainActivity创建的进程，以便于它将MainActivity组件启动起来。

这个过程共细分为35个步骤，下面进行详解：

		boolean startActivitySafely(View v, Intent intent, Object tag) {
			boolean success = false;
			try {
				success = startActivity(v, intent, tag);
			} catch (ActivityNotFoundException e) {
				Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
				Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
			}
			 	return success;
		}
		
（1）当我们点击应用程序的快捷图标的时候，Launcher组件的成员函数startActivitySafely就会被调用来启动这个应用程序的根Activity，其中，要启动的信息包含在参数intent中。

那Launcher组件是如何获取这些信息的呢：系统启动的时候，会启动一个Package管理服务，PackageManagerService，并通过它来安装系统中的应用程序，PackageManagerService在安装一个应用程序的时候，会对它的AndroidManifest.xml文件进行解析，从而得到它里面的组件信息。系统启动完成之后，就会将Launcher组件启动起来，Launcher在启动的过程中，会想PackageManagerService查询有所Action名称=“Intent.ACTION_MAIN”并且Category="Intent.CATEGORY_LAUNCHER"的Activity组件，最后为每一个Activity组件创建一个快捷图标，并将他们的信息与快捷图标关联起来。

	public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
	 if (mParent == null) {
		Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(
		this, mMainThread.getApplicationThread(), mToken, this,ntent, requestCode, options);
		if (ar != null) {
		mMainThread.sendActivityResult(
		mToken, mEmbeddedID, requestCode, ar.getResultCode(),ar.getResultData());
	 }
	...
	
（2）Instrumentation:用来监控应用程序和系统之间的交互操作，因此调用Instrumentation的execStartActivity来代为执行Activity的组件操作，以便它可以监控这个交互过程。

execStartActivity()的几个参数： 

mMainThread：类型为ActivityThead，用来描述一个应用程序进程，系统每当启动一个应用程序的时候，都会在这个应用程序里面加载一个ActivityThead类实例，这个实例保存在该进程中启动的Activity组件的父类Activity的成员变量mMainThread中。而ActivityThead的getApplicationThread()方法，用来获取它内部的一个类型为ApplicationThread的Binder本地对象。由于Launcher是Activity的子类，通过mMainThread.getApplicationThread()将Launcher所运行在的进程ActivityThead实例传递给mInstrumentation的方法execStartActivity()以便于可以将它传递给ActivityManagerService，这样ActivityManagerService接下来就可以通知Launcher组件进入Paused状态了；

mToken：类型为IBinder，是一个Binder的代理对象，指向了一个ActivityManagerService中一个类型为ActivityRecord的Binder本地对象。每个启动的Activity组件在ActivityManagerService中都有一个对应的ActivityRecord对象，用来维护对应的Activity组件的运行状态以及信息。




