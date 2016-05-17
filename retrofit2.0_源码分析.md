###Retrofit2.0 源码分析

>Retrofit 接口实现实例：

	compile 'com.squareup.retrofit2:retrofit:2.0.2'

实现方式：

	public interface IUserBiz
	{
    	@GET("users")
    	Call<List<User>> getUsers();
	}
	
通过Retrofit完成上述请求：

	Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("http://192.168.31.242:8080/springmvc_users/user/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();
		IUserBiz userBiz = retrofit.create(IUserBiz.class);
		Call<List<User>> call = userBiz.getUsers();
        call.enqueue(new Callback<List<User>>()
        {
            @Override
            public void onResponse(Call<List<User>> call, Response<List<User>> response)
            {
                Log.e(TAG, "normalGet:" + response.body() + "");
            }

            @Override
            public void onFailure(Call<List<User>> call, Throwable t)
            {

            }
        });
        
 实现原理：动态代理：
 
 Java动态代理的实现：
 
 	 public interface ITest {

        @GET("/HEHE")
        public void add(int a, int b);

    }

    public static void main(String[] args) {

        ITest iTest = (ITest) Proxy.newProxyInstance(ITest.class.getClassLoader(), new Class<?>[]{ITest.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

                Integer a = (Integer) args[0];
                Integer b = (Integer) args[1];
                System.out.println("方法名: " + method.getName());
                System.out.println("参数: " + a + " , " + b);



                GET get = method.getAnnotation(GET.class);
                System.out.println("注解:" + get.value());

                return null;
            }
        });

        iTest.add(3,5);
        }
        
  可以看到，当调用接口的任何方法时：都会调用InvocationHandler#invoke方法，这个方法可以拿到传入的参数和注解等，Retrofit也是如此，看Retrofit源码：
  
  	public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
        }
        
 Retrofit的整体实现流程
 
 通过构造者模式进行构建Retrofit对象，
 	
 	public Builder() {
      this(Platform.get());
    }
 	
 	public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
 
 * baseUrl必须指定，这个理所当然
 * 如果不着急设置callFactory，则默认直接new OkHttpClient() ,可见如果需要对OkHttpClient进行详细设置，需要构建好以后传入
 * callbackExecutor 用来将回调传递到UI线程，这里设计的比较巧妙，利用platform对象对平台进行判断，判断主要是利用Class.forName(" ")进行查找，如果是Android平台，会自定义一个Executor对象，并利用Looper.getMainLooper() 实例化一个handler对象，在Executor通过handler.post(Runnable),将回调传递到UI线程
 * 然后是adapterFactories，这里可以传入RxJavaCallAdapterFactory，使用RxJava和Retrofit相结合
 * 最后是 Converter.Factory，用于转化数据，例如将返回的reponseBody转化为对象
 
具体Call的构建流程

	return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
        
 主要就是三行代码：
 
 	ServiceMethod serviceMethod = loadServiceMethod(method);
 	
将我们的method包装成为ServiceMethod

	OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
	
通过ServiceMethod和方法的参数构造retrofit2.OkHttpCall对象

	return serviceMethod.callAdapter.adapt(okHttpCall);
	
通过serviceMethod.callAdapter.adapt()方法，将OkHttpCall进行代理包装；

逐一介绍：

ServiceMethod包含了将method转化为Call的所有信息

	
	ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result; 
    }
    
	public ServiceMethod build() {
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
      }
      
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }

      if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
      }

      return new ServiceMethod<>(this);
    }
    
首先：
    
 	callAdapter = createCallAdapter();
 	
 最终拿到的是在retrofit build()里面 adapterFactories时添加的 即为：new ExecutorCallbackCall<>(callbackExecutor, call)，该ExecutorCallbackCall唯一做的事情就是将原本call的回调转发至UI线程。

	

	
  
  
 
 
 