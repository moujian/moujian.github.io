---
 layout: post
 title: Tomcat Classloader 分析
---


### JDK的classloader
主要有Bootstrap loader, ExtClassLoader, AppClassLoader.

* Bootstrap loader使用c++实现的，主要负责加载jdk lib里面的jar.
* ExtClassLoader 是java实现的，sun.misc.Launcher$ExtClassLoader，它主要负责加载%JAVA_HOME%/jre/lib/ext里面的所有class以及java.ext.dirs系统变量指定的路径中类库
* AppClassLoader 也是java实现的，sun.misc.Launcher$AppClassLoader，它主要负责加载classpath下面的所有class或者jar. 它也有时被称为System classLoader,因为ClassLoader.getSystemClassLoader()就是返回的它

他们的关系就是Bootstrap loader ← Ext classLoader ← AppClassLoader

### Parent delegation model
即双亲委托机制, 具体来说classloader load class的时候总是先去委托parent classloader来load,当没有load成功才回自己尝试去load。具体可参见ClassLoader的实现

```java
protected synchronized Class<?> loadClass(String name, boolean resolve) 
throws ClassNotFoundException{
	// First, check if the class has already been loaded
	Class c = findLoadedClass(name);
	if (c == null) {
	    try {
		if (parent != null) {
		    c = parent.loadClass(name, false);
		} else {
		    c = findBootstrapClassOrNull(name);
		}
	    } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
		}
		if (c == null) {
	        // If still not found, then invoke findClass in order
	        // to find the class.
	        c = findClass(name);
	    }
	}
	if (resolve) {
	    resolveClass(c);
	}
	return c;
}
```

我们自定义Class Loader都需要继承ClassLoader, 并且通常只需要实现protected Class<?> findClass(String name)方法，而不需要override loadeClass方法，这里就是一个TemplateMethod设计模式。但是当然后很多场景就不遵循标准的ParentDelegation Model, 而去override这个方法，比如本文会讲到的Tomcat Class loading机制。另外注意loadClass方法通过synchronized来保证线程安全。

### Tomcat class loader

从Tomcat的入口类Bootstrap 可以看到main方法里面实例化了Bootstrap对象并且调用了Bootstrap.init()。而init里面会去初始化classLoader

```java
private void initClassLoaders() {

    commonLoader = createClassLoader("common", null);
    if( commonLoader == null ) {
        // no config file, default to this loader - we might be in a 'single' env.
        commonLoader=this.getClass().getClassLoader();
    }
    catalinaLoader = createClassLoader("server", commonLoader);
    sharedLoader = createClassLoader("shared", commonLoader);
}
```

这里面会创建三个classLoader,分别是commonLoader, catalinaLoader和sharedLoader。并且commomLoader会作为sharedLoader和catalinaLoader的parent loader.


```java
private ClassLoader createClassLoader(String name, ClassLoader parent)
        throws Exception {
        //从conf/catalina.properties读取
        String value = CatalinaProperties.getProperty(name + ".loader");
        if ((value == null) || (value.equals("")))
            return parent;
        //替换掉里面的$catlina_HOme和$catalina_base
        value = replace(value);

        List<Repository> repositories = new ArrayList<Repository>();

        StringTokenizer tokenizer = new StringTokenizer(value, ",");
        while (tokenizer.hasMoreElements()) {
    		...
    	}
    	//创建的是一个URLClassLoader
        ClassLoader classLoader = ClassLoaderFactory.createClassLoader
            (repositories, parent);

        return classLoader;

    }
```
而当我们打开$CATALINE_HOME/conf/catalina.properties会发现里面就只定义了common.loader

common.loader=${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar

所以tomcat默认配置下运行时,sharedLoader和catalinaLoader都是commonLoader. 注意其实上面catalinaLoader就是server.loader

OK 我们继续从头开始。当我们启动Tomcat的适合其实是运行的Bootstrap.main()方法。并且传入start参数。BootStrap会在init好上面的classLoader之后用catalina.loader去加载Catalina类，并且用反射调用Catalina.setParentClassLoader(sharedLoader), 这里需要注意的是代码里面会把catalinaLoader作为当前线程的ContextClassLoader.

```java
 public void init()
        throws Exception
    {

        // Set Catalina path
        setCatalinaHome();
        setCatalinaBase();

        initClassLoaders();

        Thread.currentThread().setContextClassLoader(catalinaLoader);

        SecurityClassLoad.securityClassLoad(catalinaLoader);

        
        Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
        Object startupInstance = startupClass.newInstance();

        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
        Method method = startupInstance.getClass().getMethod(methodName, paramTypes);
        method.invoke(startupInstance, paramValues);

        catalinaDaemon = startupInstance;

    }
```
我们知道Tomcat会解析server.xml并且实例化相应的Class，那这些Class是怎么load进来的呢。







