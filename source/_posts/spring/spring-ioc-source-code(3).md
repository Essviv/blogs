---
title: spring IoC源码解析(3)
author: Essviv
date: 2017-04-13 09:38:00+0800
tags: 
	- spring
	- IoC
	- createBean
---

## 2.4 createBean()方法的实现

在上面的解读中，我们知道getBean()方法的实现过程，首先尝试从缓存中获取，缓存获取失败后则尝试创建对象，在创建对象时首先检查依赖关系，在满足了依赖关系后就会调用createBean()对象获取实例. 那createBean()方法又是如何实现的呢？createBean()底层调用了doCreateBean()方法, 后者的实现大体上可以分成几个步骤:

- 调用createBeanInstance创建对象实例
- 调用populateBean()方法填充对象
- 调用initilizingBean()方法初始化对象

接下来就来看看这三个步骤的实现细节. 

### 2.4.1 createBeanInstance()方法

调用createBeanInstance()方法创建实例的实现可分为两种，一种是通过工厂对象来创建，一种是直接调用对象的构造方法来创建. 

#### 2.4.1.1 使用工厂对象创建实例

首先先来看看使用工厂对象来实例化对象的代码, 这个部分主要是调用了instantiateUsingFactoryMethod()方法来实现的，这个方法的实现代码非常复杂，总得来讲，可分为几个步骤：

1. 确定工厂对象的工厂方法factoryMethod及其参数
2. 根据第1步中确定的工厂方法和参数实例化对象

容器首先会尝试从bean定义的消息中获取定义的工厂方法factoryMethodToUse和方法参数argToUse，这里首先会查看mbd中是否已经能解析过的构造函数和方法，如果有则继续搜索是否有解析过的构造参数，如果没有则调用resolvePreparedArguments()方法进行解析. 

```java
if (explicitArgs != null) {
  argsToUse = explicitArgs;
}
else {
  Object[] argsToResolve = null;
  synchronized (mbd.constructorArgumentLock) {
    factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
    if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
      // Found a cached factory method...
      argsToUse = mbd.resolvedConstructorArguments;
      if (argsToUse == null) {
        argsToResolve = mbd.preparedConstructorArguments;
      }
    }
  }
  if (argsToResolve != null) {
    argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve);
  }
}
```

resolvePreparedArguments()方法的作用就是mbd中定义的工厂方法的参数信息，它通过遍历每个参数并判断参数的类型完成解析操作，其中最重要的是当参数是AutowiredArgumentMarker类型时，它会调用resolveAutowireArgument()方法完成参数的注入操作，关于这个方法会在后续的解读中进行详述，这里只需要知道它是用来解析参数并完成自动注入的即可.

```java
private Object[] resolvePreparedArguments(
  String beanName, RootBeanDefinition mbd, BeanWrapper bw, Member methodOrCtor, Object[] argsToResolve) {

  Class<?>[] paramTypes = (methodOrCtor instanceof Method ?
                           ((Method) methodOrCtor).getParameterTypes() : ((Constructor<?>) methodOrCtor).getParameterTypes());
  TypeConverter converter = (this.beanFactory.getCustomTypeConverter() != null ?
                             this.beanFactory.getCustomTypeConverter() : bw);
  BeanDefinitionValueResolver valueResolver =
    new BeanDefinitionValueResolver(this.beanFactory, beanName, mbd, converter);
  Object[] resolvedArgs = new Object[argsToResolve.length];
  for (int argIndex = 0; argIndex < argsToResolve.length; argIndex++) {
    Object argValue = argsToResolve[argIndex];
    MethodParameter methodParam = MethodParameter.forMethodOrConstructor(methodOrCtor, argIndex);
    GenericTypeResolver.resolveParameterType(methodParam, methodOrCtor.getDeclaringClass());
    if (argValue instanceof AutowiredArgumentMarker) {
      //这里完成了自动注入参数的解析
      argValue = resolveAutowiredArgument(methodParam, beanName, null, converter);
    }
    else if (argValue instanceof BeanMetadataElement) {
      argValue = valueResolver.resolveValueIfNecessary("constructor argument", argValue);
    }
    else if (argValue instanceof String) {
      argValue = this.beanFactory.evaluateBeanDefinitionString((String) argValue, mbd);
    }
    Class<?> paramType = paramTypes[argIndex];
    try {
      resolvedArgs[argIndex] = converter.convertIfNecessary(argValue, paramType, methodParam);
    }
    catch (TypeMismatchException ex) {
      throw new UnsatisfiedDependencyException(
        mbd.getResourceDescription(), beanName, new InjectionPoint(methodParam),
        "Could not convert argument value of type [" + ObjectUtils.nullSafeClassName(argValue) +
        "] to required type [" + paramType.getName() + "]: " + ex.getMessage());
    }
  }
  return resolvedArgs;
}
```

在经过上述的解析操作后，只要factoryMethodToUse和areToUse参数中有一个是缺失的，那么容器就需要遍历所有可能的方法来寻找相应的工厂方法和参数，寻找候选方法的过程是获取工厂对象中声明的所有方法，然后找到其中名称与工厂方法factoryMethod属性一致的候选方法，

```java
Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
List<Method> candidateSet = new ArrayList<Method>();
for (Method candidate : rawCandidates) {
  if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
    candidateSet.add(candidate);
  }
}
```

在得到候选方法之后，就需要对这些候选方法进行排序，排序的规则是public方法先于其它类型方法，参数个数多的优先.

```java
Method[] candidates = candidateSet.toArray(new Method[candidateSet.size()]);
AutowireUtils.sortFactoryMethods(candidates);

//候选方法排序
public static void sortFactoryMethods(Method[] factoryMethods) {
  Arrays.sort(factoryMethods, new Comparator<Method>() {
    @Override
    public int compare(Method fm1, Method fm2) {
      boolean p1 = Modifier.isPublic(fm1.getModifiers());
      boolean p2 = Modifier.isPublic(fm2.getModifiers());
      if (p1 != p2) {
        return (p1 ? -1 : 1);
      }
      int c1pl = fm1.getParameterTypes().length;
      int c2pl = fm2.getParameterTypes().length;
      return (c1pl < c2pl ? 1 : (c1pl > c2pl ? -1 : 0));
    }
  });
}
```

排序完后，就需要得到这些方法中匹配度最高的那个方法，匹配度的规则为类型匹配，具体规则可以参阅[这里](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/util/MethodInvoker.html#getTypeDifferenceWeight-java.lang.Class:A-java.lang.Object:A-). 如果存在两个匹配度完全一样的候选方法，那么会抛出异常. 

```java
for (Method candidate : candidates) {
  Class<?>[] paramTypes = candidate.getParameterTypes();

  //参数个数不能少于指定的参数个数
  if (paramTypes.length >= minNrOfArgs) {
    ArgumentsHolder argsHolder;

    //没有指定explictArgs的场景，需要通过类型转换及自动注入等机制获取方法的参数
    if (resolvedValues != null) {
      String[] paramNames = null;
        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
        if (pnd != null) {
          paramNames = pnd.getParameterNames(candidate);
        }
        argsHolder = createArgumentArray(
          beanName, mbd, resolvedValues, bw, paramTypes, paramNames, candidate, autowiring);
    }
    else { //显式给定了参数，则直接比较参数个数，并包装相应的参数
      // Explicit arguments given -> arguments length must match exactly.
      if (paramTypes.length != explicitArgs.length) {
        continue;
      }
      argsHolder = new ArgumentsHolder(explicitArgs);
    }

    //比较权重, 找到权重值最小的那个候选方法
    int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                          argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
    // Choose this factory method if it represents the closest match.
    if (typeDiffWeight < minTypeDiffWeight) {
      factoryMethodToUse = candidate;
      argsHolderToUse = argsHolder;
      argsToUse = argsHolder.arguments;
      minTypeDiffWeight = typeDiffWeight;
      ambiguousFactoryMethods = null;
    }
    // Find out about ambiguity: In case of the same type difference weight
    // for methods with the same number of parameters, collect such candidates
    // and eventually raise an ambiguity exception.
    // However, only perform that check in non-lenient constructor resolution mode,
    // and explicitly ignore overridden methods (with the same parameter signature).
    else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
             !mbd.isLenientConstructorResolution() &&
             paramTypes.length == factoryMethodToUse.getParameterTypes().length &&
             !Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
      if (ambiguousFactoryMethods == null) {
        ambiguousFactoryMethods = new LinkedHashSet<Method>();
        ambiguousFactoryMethods.add(factoryMethodToUse);
      }
      ambiguousFactoryMethods.add(candidate);
    }
  }
}
```

在遍历完所有的候选方法后，正常情况下就应该能获取到唯一的工厂方法factoryMethodToUse，但也不排除可能会有多个方法同时满足条件，因此这里需要再对遍历后的结果进行判断.

```java
//工厂方法仍然为空，这时候容器类无法继续实例化对象，开始收集对象信息，并抛出异常
if (factoryMethodToUse == null) {
  if (causes != null) {
    UnsatisfiedDependencyException ex = causes.removeLast();
    for (Exception cause : causes) {
      this.beanFactory.onSuppressedException(cause);
    }
    throw ex;
  }
  List<String> argTypes = new ArrayList<String>(minNrOfArgs);
  if (explicitArgs != null) {
    for (Object arg : explicitArgs) {
      argTypes.add(arg != null ? arg.getClass().getSimpleName() : "null");
    }
  }
  else {
    Set<ValueHolder> valueHolders = new LinkedHashSet<ValueHolder>(resolvedValues.getArgumentCount());
    valueHolders.addAll(resolvedValues.getIndexedArgumentValues().values());
    valueHolders.addAll(resolvedValues.getGenericArgumentValues());
    for (ValueHolder value : valueHolders) {
      String argType = (value.getType() != null ? ClassUtils.getShortName(value.getType()) :
                        (value.getValue() != null ? value.getValue().getClass().getSimpleName() : "null"));
      argTypes.add(argType);
    }
  }
  String argDesc = StringUtils.collectionToCommaDelimitedString(argTypes);
  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                  "No matching factory method found: " +
                                  (mbd.getFactoryBeanName() != null ?
                                   "factory bean '" + mbd.getFactoryBeanName() + "'; " : "") +
                                  "factory method '" + mbd.getFactoryMethodName() + "(" + argDesc + ")'. " +
                                  "Check that a method with the specified name " +
                                  (minNrOfArgs > 0 ? "and arguments " : "") +
                                  "exists and that it is " +
                                  (isStatic ? "static" : "non-static") + ".");
}
//工厂方法不能返回void值
else if (void.class == factoryMethodToUse.getReturnType()) {
  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                  "Invalid factory method '" + mbd.getFactoryMethodName() +
                                  "': needs to have a non-void return type!");
}
//不止有一个工厂方法满足条件
else if (ambiguousFactoryMethods != null) {
  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                  "Ambiguous factory method matches found in bean '" + beanName + "' " +
                                  "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
                                  ambiguousFactoryMethods);
}

//缓存mdb与factoryMethodToUse的关系，后续就可以直接从缓存中获取
if (explicitArgs == null && argsHolderToUse != null) {
  argsHolderToUse.storeCache(mbd, factoryMethodToUse);
```

如果以上的检查没有抛出任何异常，那么说明容器已经找到当前对象的工厂方法及相应的参数，可以开始创建实例的操作了. 这里需要注意的是beanFactory容器并不是自行实例化对象，而是通过InstantiationStrategy接口来完成的，默认的实现是CglibSubclassInstantiateStrategy, 也就是通过Cglib继承子类的方式来创建对象，这也是Spring框架中非常值得学习的一点，面向接口编程，提供最大限度地灵活性. 到此为止，容器通过工厂对象创建实例的操作就算全部完成了.

```java
try {
  Object beanInstance;

  if (System.getSecurityManager() != null) {
    final Object fb = factoryBean;
    final Method factoryMethod = factoryMethodToUse;
    final Object[] args = argsToUse;
    beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
      @Override
      public Object run() {
        return beanFactory.getInstantiationStrategy().instantiate(
          mbd, beanName, beanFactory, fb, factoryMethod, args);
      }
    }, beanFactory.getAccessControlContext());
  }
  else {
    //通过InstantiateStrategy接口来完成实例化工作
    beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
      mbd, beanName, this.beanFactory, factoryBean, factoryMethodToUse, argsToUse);
  }

  if (beanInstance == null) {
    return null;
  }
  bw.setBeanInstance(beanInstance);
  return bw;
}
catch (Throwable ex) {
  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                  "Bean instantiation via factory method failed", ex);
}
```

#### 2.4.1.2 直接实例化对象

创建实例对象的另一种方式就是直接通过bean对象的构造函数. 可以看到，Spring框架在尝试通过构造函数实例化对象时会根据传入的参数是否为空，是否有解析过的构造函数，是否有已经解析过的构造函数参数来决定具体调用的实例化方法.

- 当传入的构造方法参数不为空时，容器会先调用detemineConstructorsFromBeanPostProcessors方法来获取可用的构造函数, 并最终调用autowireConstructor()方法来实例化对象
- 当传入的参数为空时，会判断是否已经有解析过的构造方法，如果有，则继续判断是否有解析过的方法参数，并根据情况选择autowireConstructor()方法还是instantiateBean()方法
- 如果所有的判断条件都不满足，则直接调用instantiateBean()方法来实例化对象

```java
// Shortcut when re-creating the same bean...
boolean resolved = false;
boolean autowireNecessary = false;
if (args == null) {
  synchronized (mbd.constructorArgumentLock) {
    if (mbd.resolvedConstructorOrFactoryMethod != null) {
      resolved = true;
      autowireNecessary = mbd.constructorArgumentsResolved;
    }
  }
}
if (resolved) {
  if (autowireNecessary) {
    return autowireConstructor(beanName, mbd, null, null);
  }
  else {
    return instantiateBean(beanName, mbd);
  }
}

// Need to determine the constructor...
Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
if (ctors != null ||
    mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
    mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
  return autowireConstructor(beanName, mbd, ctors, args);
}

// No special handling: simply use no-arg constructor.
return instantiateBean(beanName, mbd);
```

通过以上的分析可以知道，Spring框架是通过instantiateBean()方法和autowireConstructor()方法来进行实例化操作. 现在就来具体看看这两个方法的作用与实现. 

#### autowireConstrutor()方法

根据autowireConstrutor文档的说明，这个方法是用来解决"构造函数自动注入"的地方, 它能够自动完成构造函数参数的自动注入. 查看这个方法的源码会发现，它的实现与上一小节“使用工厂对象创建实例”基本一样，都是先确定使用的构造方法及参数，选择候选的构造方法、排序、匹配等规则也完全一样，这里就不再做详述，有兴趣的读者可以参阅这部分源码或参考上一小节的解读.

#### instantiateBean()方法

这个方法的实现相对简单，通过InstantiateStrategy接口完成实例创建即可.

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
  try {
    Object beanInstance;
    final BeanFactory parent = this;
    if (System.getSecurityManager() != null) {
      beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
          return getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
      }, getAccessControlContext());
    }
    else {
      beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    }
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
  }
}
```

到此为止，不管是通过工厂对象创建，还是直接通过构造函数创建，Spring容器已经完成了createBean()方法中的第一步，创建BeanWrapper的操作. 有了实例对象后，就需要对实例的属性进行填充及初始化操作. 这部分内容将会在下一篇文章中进行详细阐述. 

### 2.4.2 populateBean()方法

在创建完bean对象之后，就需要填充对象的属性，这个操作由populateBean()方法完成. populateBean()方法可以分成四个步骤：

* 调用InstantiationAwareBeanPostProcessor接口的afterInstantiation()方法，这个接口定义了三个方法beforeInstantiazion, afterInstantiation以及postProcessorPropertyValues()， 分别用于在实例化对象前后以及在设置对象的属性值之前进行处理. beforeInstantiation()方法在调用createBean()方法之前就已经被调用过了，这里是在实例化对象之后，所以调用的是afterInstantiation()方法，而postProcessorPropertyValues()方法会在解析完属性值之后，填充到对象之前被调用，在populateBean()方法的第三步会碰到. 
* 根据bean对象的定义，选择合适的自动注入模式(byName或byType), 并解析相应的参数
* 调用InstantiationAwareBeanPostProcessor接口的postProcessorPropertyValues()方法，并检查当前对象的所有属性值是否已经解析完成
* 填充属性值

#### 2.4.2.1 调用postProcessAfterInstantiation()方法

可以看到，这里的实现就是获取容器中定义的所有InstantiationAwareBeanPostProcessor接口，并依次调用它们的postProcessAfterInstantiation()方法即可，没有太多复杂的逻辑，不作赘述. 

````java
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
  for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
      InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
      if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
        continueWithPropertyPopulation = false;
        break;
      }
    }
  }
}
````

#### 2.4.2.2 根据自动注入模式获取属性值

bean对象的自动注入模式是在定义bean时通过autowire属性来设置的，可选的值包括byName, byType以及constructor, 其中constructor的情况已经在创建对象实例时处理过了，因此这里只需要考虑byName和byType两种情况. 顾名思义，byName就是通过属性的名称与容器中定义的bean名称进行自动注入，而byType则是根据类型注入.

````java
if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
    mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
  MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

  // Add property values based on autowire by name if applicable.
  if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
    autowireByName(beanName, mbd, bw, newPvs);
  }

  // Add property values based on autowire by type if applicable.
  if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
    autowireByType(beanName, mbd, bw, newPvs);
  }

  pvs = newPvs;
}
````

#### 根据bean名称自动注入

从autowireByName()的实现可以看出，这个模式的自动注入过程比较简单，首先获取当前bean对象中还没有被设置过的非简单类型的属性(Spring框架不允许简单类型的自动注入), 然后遍历这些属性，根据属性的名称尝试从容器中获取相应的bean对象，得到依赖的对象后，记录该属性对应的值，并设置相应的依赖关系表即可. 

````java
protected void autowireByName(
  String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

  String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
  for (String propertyName : propertyNames) {
    if (containsBean(propertyName)) {
      Object bean = getBean(propertyName);
      pvs.add(propertyName, bean);
      registerDependentBean(propertyName, beanName);
    }
  }
}
````

#### 根据类型自动注入



#### 2.4.2.3 调用postProcessPropertyValues()并检查依赖

#### 2.4.2.4 设置属性值

### 2.4.3 initilizeBean()方法

在调用populateBean()方法填充完bean对象后，对象的创建工作就基本完成了. Spring容器中定义了定义了生命周期方法，用于在构造完对象之后和析构对象之前执行定制化的操作，其中包括InitializingBean接口，Aware接口以及配置文件中的init-method属性. 这些属性的处理都是在initializeBean()方法中进行处理的.

可以看到， initializeBean()方法主要是四个步骤：

* 调用Aware相关的接口，其中包括BeanNameAware接口，BeanClassLoaderAware接口以及BeanFactoryAware接口
* 调用BeanPostProcessor的postProcessorBeforeInitialization()方法
* 调用initMethod方法，包括InitializingBean接口以及init-method属性定义的方法, 其中InitializingBean接口先于init-method方法执行
* 调用BeanPostProcessor的postProcessorAfterInitialization()方法

````java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
  //执行Aware接口方法
  invokeAwareMethods(beanName, bean);
  
  //执行BeanPostProcessor的beforeIntialization()方法
  Object wrappedBean = bean;
  if (mbd == null || !mbd.isSynthetic()) {
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  }

  //执行init()方法
  try {
    invokeInitMethods(beanName, wrappedBean, mbd);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
      (mbd != null ? mbd.getResourceDescription() : null),
      beanName, "Invocation of init method failed", ex);
  }

  //执行BeanPostProcessor接口的afterBeanInitialization方法
  if (mbd == null || !mbd.isSynthetic()) {
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }
  return wrappedBean;
}

//invokeAwareMethod
private void invokeAwareMethods(final String beanName, final Object bean) {
  if (bean instanceof Aware) {
    if (bean instanceof BeanNameAware) {
      ((BeanNameAware) bean).setBeanName(beanName);
    }
    if (bean instanceof BeanClassLoaderAware) {
      ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
    }
    if (bean instanceof BeanFactoryAware) {
      ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
    }
  }
}

//invokeInitMethod
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
  throws Throwable {

  //执行InitializaingBean接口
  boolean isInitializingBean = (bean instanceof InitializingBean);
  if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {    
   ((InitializingBean) bean).afterPropertiesSet();
  }

  //执行init-method方法
  if (mbd != null) {
    String initMethodName = mbd.getInitMethodName();
    if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
        !mbd.isExternallyManagedInitMethod(initMethodName)) {
      invokeCustomInitMethod(beanName, bean, mbd);
    }
  }
}
````

