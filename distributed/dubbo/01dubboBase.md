- 集群容错
默认3次，然后进行轮询

- dubbo spi
与javaSpi类似，但是可以切换不同的实现类，配置文件为key-value形式

- URL

- Invoker

- dubbo aop及IOC
1. 首先是普通写法实现blackCar其接口为Car
2. CarWrapper也实现Car，但其有个属性为Car，且无参构造变为1参构造(此1参会被注入blackCar)，再把该类添加到SPI配置文件中(只需value)，此时从容器中获取到的blackCar就变为CarWrapper
```java
public class CarWrapper implements Car {
	Car car;
	public CarWrapper (Car car) {
		this.car = car;
	}
}
```
```java
ExtensionLoader<Car> extensionLoader= ExtensionLoader.getExtensionLoader(Car.class);
Car car = extensionLoader.getExtension("black")
```

## 4.org.apache.dubbo.common.extension.ExtensionLoader#getExtensionLoader
```java
// 左边是类型，右边是扩展类
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }

    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
    	// 如果没有则生成一个包装类
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

## 5.org.apache.dubbo.common.extension.ExtensionLoader#getExtension(java.lang.String)
```java
public T getExtension(String name) {
    return getExtension(name, true);
}
```
```java
public T getExtension(String name, boolean wrap) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    // 判断是否有默认类在接口@SPI注解设定
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    // 返回一个holder保证无论如何有对象返回，方便下一步的并发锁
    final Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
            	// 创建扩展点对象
                instance = createExtension(name, wrap);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```
- org.apache.dubbo.common.extension.ExtensionLoader#createExtension
```java
private T createExtension(String name, boolean wrap) {
	// 根据<name,class>获取class
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null || unacceptableExceptions.contains(name)) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
        	// 反射实例化类
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.getDeclaredConstructor().newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // IOC
        injectExtension(instance);


        if (wrap) {
        	// AOP
            List<Class<?>> wrapperClassesList = new ArrayList<>();
            if (cachedWrapperClasses != null) {
                wrapperClassesList.addAll(cachedWrapperClasses);
                wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                Collections.reverse(wrapperClassesList);
            }
            // 遍历所有的包装类
            if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                for (Class<?> wrapperClass : wrapperClassesList) {
                	// 判断是不是包装类根据注解、名称
                    Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                    if (wrapper == null
                            || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), name))) {
                        // 调用单参构造函数反射实例化，同时对该对象进行递归IOC
                        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                    }
                }
            }
        }
        // 生命周期回调
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

## 5.1org.apache.dubbo.common.extension.ExtensionLoader#getExtensionClasses
```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
        	// 加载配置文件
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```
- org.apache.dubbo.common.extension.ExtensionLoader#loadExtensionClasses
```java
private Map<String, Class<?>> loadExtensionClasses() {
    cacheDefaultExtensionName();

    Map<String, Class<?>> extensionClasses = new HashMap<>();

    for (LoadingStrategy strategy : strategies) {
        loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
        loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"), strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
    }

    return extensionClasses;
}
```
- org.apache.dubbo.common.extension.ExtensionLoader#loadDirectory
```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
                               boolean extensionLoaderClassLoaderFirst, boolean overridden, String... excludedPackages) {
    String fileName = dir + type;
    try {
        Enumeration<java.net.URL> urls = null;
        ClassLoader classLoader = findClassLoader();

        // try to load from ExtensionLoader's ClassLoader first
        if (extensionLoaderClassLoaderFirst) {
            ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
            if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                urls = extensionLoaderClassLoader.getResources(fileName);
            }
        }

        if (urls == null || !urls.hasMoreElements()) {
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
        }

        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                // 解析文件
                loadResource(extensionClasses, classLoader, resourceURL, overridden, excludedPackages);
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", description file: " + fileName + ").", t);
    }
}
```
- org.apache.dubbo.common.extension.ExtensionLoader#loadResource
```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader,
                              java.net.URL resourceURL, boolean overridden, String... excludedPackages) {
    try {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
            String line;
            String clazz = null;
            while ((line = reader.readLine()) != null) {
            	// 处理注释逻辑
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        // 处理=号逻辑
                        int i = line.indexOf('=');
                        if (i > 0) {
                            name = line.substring(0, i).trim();
                            clazz = line.substring(i + 1).trim();
                        } else {
                            clazz = line;
                        }
                        if (StringUtils.isNotEmpty(clazz) && !isExcluded(clazz, excludedPackages)) {
                            loadClass(extensionClasses, resourceURL, Class.forName(clazz, true, classLoader), name, overridden);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                        exceptions.put(line, e);
                    }
                }
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", class file: " + resourceURL + ") in " + resourceURL, t);
    }
}
```
- org.apache.dubbo.common.extension.ExtensionLoader#loadClass
```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name,
                           boolean overridden) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    // 判断Adaptive
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        cacheAdaptiveClass(clazz, overridden);
        //  clazz.getConstructor(type)
        // 判断是不是Wrapper，即是否有一个有参构造
    } else if (isWrapperClass(clazz)) {
        cacheWrapperClass(clazz);
    } else {
    	// 判断无参构造
        clazz.getConstructor();
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }

        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                cacheName(clazz, n);
                // 保存
                saveInExtensionClass(extensionClasses, clazz, n, overridden);
            }
        }
    }
}
```

### 5.2org.apache.dubbo.common.extension.ExtensionLoader#injectExtension
```java
private T injectExtension(T instance) {

    if (objectFactory == null) {
        return instance;
    }

    try {
        for (Method method : instance.getClass().getMethods()) {
            if (!isSetter(method)) {
                continue;
            }
            /**
             * Check {@link DisableInject} to see if we need auto injection for this property
             */
            // @DisableInject表示无需注入
            if (method.getAnnotation(DisableInject.class) != null) {
                continue;
            }
            // 获取set方法参数类型
            Class<?> pt = method.getParameterTypes()[0];
            if (ReflectUtils.isPrimitives(pt)) {
                continue;
            }

            try {
            	// 得到setXxx的xxx
                String property = getSetterProperty(method);
                /*
                private ExtensionLoader(Class<?> type) {
				    this.type = type;
				    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
				}
                */
                // objectFactory = AdaptiveExtensionFactory,因为该类上有@Adaptive
                // 从SPI和spring中找值赋值
                Object object = objectFactory.getExtension(pt, property);
                if (object != null) {
                    method.invoke(instance, object);
                }
            } catch (Exception e) {
                logger.error("Failed to inject via method " + method.getName()
                        + " of interface " + type.getName() + ": " + e.getMessage(), e);
            }

        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```
#### 5.2.1org.apache.dubbo.common.extension.factory.AdaptiveExtensionFactory#getExtension
```java
public <T> T getExtension(Class<T> type, String name) {
	// 此处会获得其它实现类SpiExtensionFactory、SpringExtensionFactory
	// 如果是SpringExtensionFactory则直接从spring容器中注入如果是SpiExtensionFactory则返回Adaptive代理类,但是需要写在接口中写如下代码才能注入
	/*
	// 如果不传Url则需要传入的类中有getUrl方法来指定具体实现类
	@Adaptive
	String getCarName(Url url)
	*/
    for (ExtensionFactory factory : factories) {
        T extension = factory.getExtension(type, name);
        if (extension != null) {
            return extension;
        }
    }
    return null;
}
```
##### 5.2.1.1org.apache.dubbo.config.spring.extension.SpringExtensionFactory#getExtension
```java
public <T> T getExtension(Class<T> type, String name) {

    //SPI should be get from SpiExtensionFactory
    if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
        return null;
    }

    for (ApplicationContext context : CONTEXTS) {
        T bean = BeanFactoryUtils.getOptionalBean(context, name, type);
        if (bean != null) {
            return bean;
        }
    }

    //logger.warn("No spring extension (bean) named:" + name + ", try to find an extension (bean) of type " + type.getName());

    return null;
}
```
##### 5.2.1.2org.apache.dubbo.common.extension.factory.SpiExtensionFactory#getExtension
```java
public <T> T getExtension(Class<T> type, String name) {
    if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
        ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
        if (!loader.getSupportedExtensions().isEmpty()) {
            return loader.getAdaptiveExtension();
        }
    }
    return null;
}
```
- org.apache.dubbo.common.extension.ExtensionLoader#getAdaptiveExtension
```java
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("Failed to create adaptive instance: " +
                    createAdaptiveInstanceError.toString(),
                    createAdaptiveInstanceError);
        }

        synchronized (cachedAdaptiveInstance) {
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                	// 核心方法
                    instance = createAdaptiveExtension();
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                }
            }
        }
    }

    return (T) instance;
}
```
- org.apache.dubbo.common.extension.ExtensionLoader#createAdaptiveExtension|生成代理对象并依赖注入
```java
private T createAdaptiveExtension() {
    try {
    	// 会判断是否有指定Adaptive，直接生成相应代码
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```