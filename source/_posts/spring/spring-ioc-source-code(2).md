---
title: spring IoC源码分析(2)
author: essviv
date: 2017-04-11 19:42:00+0800
tags:
	- spring
	- IoC
	- DI
---

## 2. 使用容器

在本文的第一部分中，我们对容器的初始化操作refresh()方法做了全面的解读，了解了容器构造的整个过程，包括读取bean对象的定义，配置BeanFactoyPostProcessor&BeanPostProcessor接口，以及其它的属性. 在第一部分的解读过程中，还遗留了一个方法，就是getBean()方法. 

getBean()方法的使用场景有以下几个，从字面上看，这个方法就是从容器中获取相应的实例对象，那它具体的实现过程又是怎么样的呢? 

* 容器初始化结束后，会调用preInstantiateSingleton()方法来实例化lazyInit属性为false的"单例"对象
* 应用程序调用getBean()方法获取对象

````java
public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        TestController testController = applicationContext.getBean(TestController.class);

        testController.test();
    }
````

查看ClassPathXmlApplicationContext类的getBean()方法，可以看到它调用的是AbstractApplicationContext类的相应实现，而AbstractApplicationContext方法中是将这个方法的实现委托给内部的BeanFactory对象来实现，从本文第一部分的解读中我们知道，ClassPathXmlApplicationContext内部的BeanFactory对象是在refresh()方法中调用obtainFreshBeanFactory()方法时生成的，默认是DefaultListableBeanFactory的实例，因此我们直接查看DefaultListableBeanFactory类的相应实现即可.

````java
@Override
public <T> T getBean(Class<T> requiredType) throws BeansException {
  assertBeanFactoryActive();
  return getBeanFactory().getBean(requiredType);
}
````

从DefaultListableBeanFactory类的实现源码可以看到，获取bean实例对象可以分成以下几个步骤：

1. 获取容器中指定类型的bean名称
2. 如果获取的bean名称超过1个，则对名称进行过滤
3. 判断过滤后的bean名称个数，并根据相应的个数获取实例对象

接下来我们来详细地看下这些步骤的执行过程. 

## 2.1 获取指定类型的bean名称

这个步骤是通过getBeanNamesForType()方法来完成的，它是获取容器中符合指定类型的bean对象的名称. 首先来看下这个方法的实现, 可以看到，方法首先会判断当前容器是否处于“冻结配置”的阶段(在容器完成refresh()操作后，配置就会处于“冻结”状态), 如果处于冻结状态，容器就会尝试从缓存中获取相应的名称. 当从缓存中无法获取相应的名称时，才会调用doGetBeanNamesForType()方法进行解析，解析完后会将结果放入缓存.

````java
public String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
  if (!isConfigurationFrozen() || type == null || !allowEagerInit) {
    return doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, allowEagerInit);
  }
  Map<Class<?>, String[]> cache =
    (includeNonSingletons ? this.allBeanNamesByType : this.singletonBeanNamesByType);
  String[] resolvedBeanNames = cache.get(type);
  if (resolvedBeanNames != null) {
    return resolvedBeanNames;
  }
  resolvedBeanNames = doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, true);
  if (ClassUtils.isCacheSafe(type, getBeanClassLoader())) {
    cache.put(type, resolvedBeanNames);
  }
  return resolvedBeanNames;
}
````

从上面的解读中可以知道，实际的工作是在doGetBeanNamesForType()方法中完成的. 这个方法的实现可以分成两个部分，一部分是遍历容器中定义的所有对象的类型，找到其中匹配的对象；另一部分是遍历自定义的单例对象，找到其中匹配的对象名称. 但是不管是遍历哪部分的对象，其实现的思路都是一致的

* 首先匹配beanName对象的类型与请求的类型是否一致
* 不一致的情况下，再尝试匹配beanName的工厂对象创建的对象类型与请求的类型是否一致

具体的匹配操作是通过isTypeMatch()方法来完成的.

````java
private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
  List<String> result = new ArrayList<String>();

  // 遍历容器中的对象
  for (String beanName : this.beanDefinitionNames) {
    RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        // Only check bean definition if it is complete.
        if (!mbd.isAbstract() && (allowEagerInit ||
                                  ((mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading())) &&
                                  !requiresEagerInitForType(mbd.getFactoryBeanName()))) {
          // In case of FactoryBean, match object created by FactoryBean.
          boolean isFactoryBean = isFactoryBean(beanName, mbd);
          boolean matchFound = (allowEagerInit || !isFactoryBean || containsSingleton(beanName)) &&
            (includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type);
          if (!matchFound && isFactoryBean) {
            // In case of FactoryBean, try to match FactoryBean instance itself next.
            beanName = FACTORY_BEAN_PREFIX + beanName;
            matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
          }
          if (matchFound) {
            result.add(beanName);
          }
        }
  }

  // 遍历程序手动注册的单例对象
  for (String beanName : this.manualSingletonNames) {
    try {
      // In case of FactoryBean, match object created by FactoryBean.
      if (isFactoryBean(beanName)) {
        if ((includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type)) {
          result.add(beanName);
          // Match found for this bean: do not match FactoryBean itself anymore.
          continue;
        }
        // In case of FactoryBean, try to match FactoryBean itself next.
        beanName = FACTORY_BEAN_PREFIX + beanName;
      }
      // Match raw bean instance (might be raw FactoryBean).
      if (isTypeMatch(beanName, type)) {
        result.add(beanName);
      }
    }
    catch (NoSuchBeanDefinitionException ex) {
      // Shouldn't happen - probably a result of circular reference resolution...
      if (logger.isDebugEnabled()) {
        logger.debug("Failed to check manually registered singleton with name '" + beanName + "'", ex);
      }
    }
  }

  return StringUtils.toStringArray(result);
}
````

isTypeMatch()方法的作用是判断提供的beanName对象是否与请求的类型匹配，在判断的过程，它会考虑beanName是否为FactoryBean，以及需要判断的类型是FactoryBean本身还是它创建的对象. 它的实现可以分成三个步骤

* 从缓存中获取实例对象

这里可以看到，当缓存中获取到对象时，它首先会判断该对象类型是否为FactoryBean，如果是的话它还需要判断是否比较的是FactoryBean本身的类型还是它创建的对象类型，这点是通过beanName来判断的. 如果不是FactoryBean, 则直接判断返回的对象类型与请求的类型是否匹配即可.

````java
Object beanInstance = getSingleton(beanName, false);
if (beanInstance != null) {
  if (beanInstance instanceof FactoryBean) {
    if (!BeanFactoryUtils.isFactoryDereference(name)) {
      Class<?> type = getTypeForFactoryBean((FactoryBean<?>) beanInstance);
      return (type != null && typeToMatch.isAssignableFrom(type));
    }
    else {
      return typeToMatch.isInstance(beanInstance);
    }
  }
  else {
    return (!BeanFactoryUtils.isFactoryDereference(name) && typeToMatch.isInstance(beanInstance));
  }
}
````

* 缓存获取失败的情况下，如果当前容器不包含bean的定义，且父容器不为空，则交给父容器判断类型匹配的工作
* 否则尝试从容器中定义的bean信息判断类型匹配信息

这里首先判断bean定义的对象是否是代理对象(dbd!=null)，如果是则判断被代理对象的类型能否满足条件，否则直接获取当前bean定义的类型. 在得到类型后，仍然需要判断当前对象是否是FactoryBean以及是否需要匹配的是FactoryBean对象本身的类型，这点和从缓存中获取到对象后的逻辑一样. 

````java
// Retrieve corresponding bean definition.
RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

Class<?> classToMatch = typeToMatch.getRawClass();
Class<?>[] typesToMatch = (FactoryBean.class == classToMatch ?
                           new Class<?>[] {classToMatch} : new Class<?>[] {FactoryBean.class, classToMatch});

// 判断当前bean定义是否是代理的对象，是的话首先判断被代理对象是否满足要求
BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
if (dbd != null && !BeanFactoryUtils.isFactoryDereference(name)) {
  RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
  Class<?> targetClass = predictBeanType(dbd.getBeanName(), tbd, typesToMatch);
  if (targetClass != null && !FactoryBean.class.isAssignableFrom(targetClass)) {
    return typeToMatch.isAssignableFrom(targetClass);
  }
}

//如果不是代理对象，获取当前bean定义的类型信息
Class<?> beanType = predictBeanType(beanName, mbd, typesToMatch);
if (beanType == null) {
  return false;
}

// 判断是否为FactoryBean以及是否需要匹配FactoryBean本身的类型
if (FactoryBean.class.isAssignableFrom(beanType)) {
  if (!BeanFactoryUtils.isFactoryDereference(name)) {
    // If it's a FactoryBean, we want to look at what it creates, not the factory class.
    beanType = getTypeForFactoryBean(beanName, mbd);
    if (beanType == null) {
      return false;
    }
  }
}
else if (BeanFactoryUtils.isFactoryDereference(name)) {
  // Special case: A SmartInstantiationAwareBeanPostProcessor returned a non-FactoryBean
  // type but we nevertheless are being asked to dereference a FactoryBean...
  // Let's check the original bean class and proceed with it if it is a FactoryBean.
  beanType = predictBeanType(beanName, mbd, FactoryBean.class);
  if (beanType == null || !FactoryBean.class.isAssignableFrom(beanType)) {
    return false;
  }
}

return typeToMatch.isAssignableFrom(beanType);
````

在上面的解读中可以看到，获取beanName对象的实际类型是通过predictBeanType()方法来实现的，这个方法最终会调用doResolveBeanClass()方法来匹配，这个方法首先会根据typesToMatch参数判断当前是否在进行类型匹配操作，并以此为依据选择类加载器classLoaderToUse, 对于类型匹配操作来讲，Spring容器会使用临时的类加载器，以避免在类型匹配的过程中加载相关的类. 在确定了要使用的类加载器后，就会使用该加载器来加载bean，并返回相应的类信息. 

````java
private Class<?> doResolveBeanClass(RootBeanDefinition mbd, Class<?>... typesToMatch) throws ClassNotFoundException {
  ClassLoader beanClassLoader = getBeanClassLoader();
  ClassLoader classLoaderToUse = beanClassLoader;
  //判断当前是否是在进行类型匹配操作
  if (!ObjectUtils.isEmpty(typesToMatch)) {
    // When just doing type checks (i.e. not creating an actual instance yet),
    // use the specified temporary class loader (e.g. in a weaving scenario).
    ClassLoader tempClassLoader = getTempClassLoader();
    if (tempClassLoader != null) {
      classLoaderToUse = tempClassLoader;
      if (tempClassLoader instanceof DecoratingClassLoader) {
        DecoratingClassLoader dcl = (DecoratingClassLoader) tempClassLoader;
        for (Class<?> typeToMatch : typesToMatch) {
          dcl.excludeClass(typeToMatch.getName());
        }
      }
    }
  }
  
  //首先从mbd中获取类名信息
  String className = mbd.getBeanClassName();
  if (className != null) {
    Object evaluated = evaluateBeanDefinitionString(className, mbd);
    if (!className.equals(evaluated)) {
      // A dynamically resolved expression, supported as of 4.2...
      if (evaluated instanceof Class) {
        return (Class<?>) evaluated;
      }
      else if (evaluated instanceof String) {
        return ClassUtils.forName((String) evaluated, classLoaderToUse);
      }
      else {
        throw new IllegalStateException("Invalid class name expression result: " + evaluated);
      }
    }
    // When resolving against a temporary class loader, exit early in order
    // to avoid storing the resolved Class in the bean definition.
    if (classLoaderToUse != beanClassLoader) {
      return ClassUtils.forName(className, classLoaderToUse);
    }
  }
  return mbd.resolveBeanClass(beanClassLoader);
}
````

## 2.2 过滤bean名称

在getBean()方法的第一个步骤中，首先根据对象的类型获取到了相应的bean名称数组，如果数组的长度超过1，那么需要对这些对象名称进行过滤. 过滤的主要思路就是去掉那些autowireCandidate属性设置为false的bean定义. 

这里有个比较容易疑惑的地方，就是在过滤的过程中，也保留了containBeanDefinition()返回false的bean名称. 查看这个方法的javaDoc可以知道，这个方法仅判断当前的BeanFactory容器中是否含有这个名称的对象定义，不会考虑它的父容器中是否有这个bean. 但过滤完后，这些bean名称都是通过getBean()方法来进一步获取具体的对象定义，而getBean()方法在当前beanFactory如果找不到这个定义，会尝试从它的父容器中寻找，因此这里在过滤的时候，需要保留那些在当前容器中并不存在的bean名称.

````java
if (beanNames.length > 1) {
  ArrayList<String> autowireCandidates = new ArrayList<String>();
  for (String beanName : beanNames) {
    if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
      autowireCandidates.add(beanName);
    }
  }
  if (autowireCandidates.size() > 0) {
    beanNames = autowireCandidates.toArray(new String[autowireCandidates.size()]);
  }
}
````

## 2.3 根据名称获取bean对象实例

在完成了第二步的过滤后，接着就会用名称从容器中查找相应的bean实例. 这里分几种情况考虑：

* 如果只有一个名称满足条件，那么直接尝试获取这个名称指定的实例就可以了
* 如果满足条件的有多个名称 ，那么需要依次按primary, prirority进行过滤，直到找到唯一的实例对象为止
* 如果没有任何名称满足条件，则尝试使用父容器获取相应的实例对象
* 否则直接抛出异常

### 2.3.1 多个名称符合条件的情况

对于第一种情况，只有一个名称满足条件，这里暂时略过，后面统一对getBean()这个方法进行详述. 而第三种情况，其实是在父容器中调用同样的方法，也直接跳过. 这里主要解读下第二种情况，从源码中可以看出，在多个名称符合条件的情况下，Spring框架首先会依次获取这些名称的对象，然后通过determinePrimaryCandidate()方法判断其中是否有**唯一**的primary属性为true的实例对象，如果有，直接返回这个对象； 如果没有找到，那么会继续调用determiHighestPriorityCandidate()方法判断是否有**唯一**的Priority注解的对象，如果有，直接返回这个对象，如果还是没有找到那么说明存在多个对象符合条件，那么就抛出异常.

````java
		if (beanNames.length == 1) {
			return getBean(beanNames[0], requiredType, args);
		}
		else if (beanNames.length > 1) {
			Map<String, Object> candidates = new HashMap<String, Object>();
			for (String beanName : beanNames) {
				candidates.put(beanName, getBean(beanName, requiredType, args));
			}
			String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
			if (primaryCandidate != null) {
				return getBean(primaryCandidate, requiredType, args);
			}
			String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
			if (priorityCandidate != null) {
				return getBean(priorityCandidate, requiredType, args);
			}
			throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
		}
		else if (getParentBeanFactory() != null) {
			return getParentBeanFactory().getBean(requiredType, args);
		}
		else {
			throw new NoSuchBeanDefinitionException(requiredType);
		}
````

首先来看看determinePrimaryCandidate()方法的实现，这里依次判断每个候选对象的primary属性，如果有超过一个对象设置primary属性为true, 那么直接抛出异常；如果只有一个对象设置了这个属性，则直接返回这个对象，根据上面的解读，如果只有唯一一个对象设置了这个属性值，那么这个对象将会被直接返回，否则返回null.

````java
	protected String determinePrimaryCandidate(Map<String, Object> candidateBeans, Class<?> requiredType) {
		String primaryBeanName = null;
		for (Map.Entry<String, Object> entry : candidateBeans.entrySet()) {
			String candidateBeanName = entry.getKey();
			Object beanInstance = entry.getValue();
			if (isPrimary(candidateBeanName, beanInstance)) {
				if (primaryBeanName != null) {
					boolean candidateLocal = containsBeanDefinition(candidateBeanName);
					boolean primaryLocal = containsBeanDefinition(primaryBeanName);
					if (candidateLocal && primaryLocal) {
						throw new NoUniqueBeanDefinitionException(requiredType, candidateBeans.size(),
								"more than one 'primary' bean found among candidates: " + candidateBeans.keySet());
					}
					else if (candidateLocal) {
						primaryBeanName = candidateBeanName;
					}
				}
				else {
					primaryBeanName = candidateBeanName;
				}
			}
		}
		return primaryBeanName;
	}

protected boolean isPrimary(String beanName, Object beanInstance) {
  if (containsBeanDefinition(beanName)) {
    return getMergedLocalBeanDefinition(beanName).isPrimary();
  }
  BeanFactory parentFactory = getParentBeanFactory();
  return (parentFactory instanceof DefaultListableBeanFactory &&
          ((DefaultListableBeanFactory) parentFactory).isPrimary(beanName, beanInstance));
}
````

另外再来看看determineHighestPriorirtyCandidate()方法. 应该来讲，这里的实现思路和上面determineByPrimaryCandidate()方法的思路完全一样，只不过判断是priority，在此基础上还增加了priority属性的比较，其它的都一样. 这里就不再赘述. 

````java
	protected String determineHighestPriorityCandidate(Map<String, Object> candidateBeans, Class<?> requiredType) {
		String highestPriorityBeanName = null;
		Integer highestPriority = null;
		for (Map.Entry<String, Object> entry : candidateBeans.entrySet()) {
			String candidateBeanName = entry.getKey();
			Object beanInstance = entry.getValue();
			Integer candidatePriority = getPriority(beanInstance);
			if (candidatePriority != null) {
				if (highestPriorityBeanName != null) {
					if (candidatePriority.equals(highestPriority)) {
						throw new NoUniqueBeanDefinitionException(requiredType, candidateBeans.size(),
								"Multiple beans found with the same priority ('" + highestPriority + "') " +
										"among candidates: " + candidateBeans.keySet());
					}
					else if (candidatePriority < highestPriority) {
						highestPriorityBeanName = candidateBeanName;
						highestPriority = candidatePriority;
					}
				}
				else {
					highestPriorityBeanName = candidateBeanName;
					highestPriority = candidatePriority;
				}
			}
		}
		return highestPriorityBeanName;
	}
````

### 2.3.2 只有一个名称符合条件的情况

到这里，我们就把所有特殊的情况都解读过了，现在只剩下最基本的情况，就是根据bean名称来获取相应的实现，这也是在getBean()方法中实现的. 在AbstractBeanFactory类中可以看到，这个方法最终调用了doGetBean()方法，它的实现过程大体可以分成三个步骤：

* 尝试从已经创建或正在创建的singleton对象中获取，如果有则直接调用getObjectForBeanInstance()返回
* 如果当前的容器中不包含有这个bean的定义且父容器不为空的情况下，尝试从父容器中获取该bean实例对象
* 尝试在当前容器中创建这个bean实例对象，这步又可以进一步细分成以下两个步骤
  * 保证当前bean对象的依赖对象已经创建完成
  * 根据当前bean对象的作用域创建相应的实例对象
* 创建完对象后，如果类型不符合，那么尝试使用类型转换服务转换成目标类型. 


接下来分别对这些步骤进行解读.

#### 2.3.2.1 尝试从缓存中获取singleton

在这一步中，BeanFactory会尝试从singletonObjects属性中读取这个beanName指定的实例对象，如果找到了说明这个对象之前已经被创建过了，直接返回这个对象即可；如果没有找到这个对象，但发现它正在被创建，那么会尝试从earlySingletonObjects属性中查找，如果还是没有找到，则尝试从singletonFactories中查找相应的对象工厂，如果有这个对象的对象工厂，则尝试创建这个对象. 

````java
// Eagerly check singleton cache for manually registered singletons.
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
  bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}

//getSingleton方法
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  Object singletonObject = this.singletonObjects.get(beanName);
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
      singletonObject = this.earlySingletonObjects.get(beanName);
      if (singletonObject == null && allowEarlyReference) {
        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          singletonObject = singletonFactory.getObject();
          this.earlySingletonObjects.put(beanName, singletonObject);
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
````

在获取到sharedInstance对象后，代码中会调用getObjectForBeanInstance()方法，这个方法的作用是对获取的对象进行解析，如果它是普通对象或者请求的是工厂对象本身(bean名称以'&'开头)则直接返回； 否则就使用工厂方法创建相应的对象. 这里的逻辑相对简单，就不做过多的阐述. 

````java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
````

#### 2.3.2.2 从父容器中获取相应的bean实例对象

如果当前容器对象不包含有这个beanName, 且父容器不为空的情况下，会尝试从父容器中获取这个bean实例

````java
// Check if bean definition exists in this factory.
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
  // Not found -> check parent.
  String nameToLookup = originalBeanName(name);
  if (args != null) {
    // Delegation to parent with explicit args.
    return (T) parentBeanFactory.getBean(nameToLookup, args);
  }
  else {
    // No args -> delegate to standard getBean method.
    return parentBeanFactory.getBean(nameToLookup, requiredType);
  }
}
````

#### 2.3.2.3 尝试从当前容器中加载bean对象

当父容器为空，或者当前容器中含有指定的bean名称的定义，那么就会在当前容器中尝试加载该对象. 加载的过程可以分成两个步骤，第一个是确保当前对象依赖的所有对象都已经创建完成，第二个是根据对象作用域的不同创建相应的对象. 

* 确保依赖的对象已经创建完成

````java
// Guarantee initialization of beans that the current bean depends on.
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
  for (String dependsOnBean : dependsOn) {
    if (isDependent(beanName, dependsOnBean)) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                      "Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
    }
    registerDependentBean(dependsOnBean, beanName);
    getBean(dependsOnBean);
  }
}
````

对于每个依赖的对象，都调用isDependent方法来确认是否存在循环依赖关系. isDependent的作用是判断第二个参数指定的对象是否依赖于第一个参数的对象，如果后者依赖于前者，而前者的dependsOn属性中又包含有第二个，说明两者之前有循环依赖关系，这时直接抛出循环依赖异常.

````java
//该方法用于判断第二个参数是否依赖于第一个参数， 第一个参数表示当前对象名称，第二个参数表示依赖于当前对象的名称
private boolean isDependent(String beanName, String dependentBeanName, Set<String> alreadySeen) {
  String canonicalName = canonicalName(beanName);
  if (alreadySeen != null && alreadySeen.contains(beanName)) {
    return false;
  }
  //dependentBeanMap属性中存储的是对象间的依赖关系，key为当前对象，value值为依赖于它的对象名称集合.
  Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
  if (dependentBeans == null) {
    return false;
  }
  
  //如果dependentBeans中包含有这个名称，说明dependentBeanName依赖于当前对象
  if (dependentBeans.contains(dependentBeanName)) {
    return true;
  }
  
  //如果denpendentBeanName不依赖于当前对象，则继续判断它传递依赖于当前对象，深度优先遍历
  for (String transitiveDependency : dependentBeans) {
    if (alreadySeen == null) {
      alreadySeen = new HashSet<String>();
    }
    alreadySeen.add(beanName);
    if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
      return true;
    }
  }
  return false;
}
````

如果不存在循环依赖关系，则调用registerDependentBean方法注册对象间的依赖关系. 这个方法会往dependentBeanMap中插入相应的名称，这个属性维护了当前对象以及依赖于这个对象的对象名称集合，另外这个方法还会往dependenciesForBeanMap属性中插入数据，这个属性维护的是当前对象以及当前对象所依赖的对象名称集合. 

````java
public void registerDependentBean(String beanName, String dependentBeanName) {
  // A quick check for an existing entry upfront, avoiding synchronization...
  String canonicalName = canonicalName(beanName);
  Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
  if (dependentBeans != null && dependentBeans.contains(dependentBeanName)) {
    return;
  }

  // No entry yet -> fully synchronized manipulation of the dependentBeans Set
  synchronized (this.dependentBeanMap) {
    dependentBeans = this.dependentBeanMap.get(canonicalName);
    if (dependentBeans == null) {
      dependentBeans = new LinkedHashSet<String>(8);
      this.dependentBeanMap.put(canonicalName, dependentBeans);
    }
    dependentBeans.add(dependentBeanName);
  }
  synchronized (this.dependenciesForBeanMap) {
    Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);
    if (dependenciesForBean == null) {
      dependenciesForBean = new LinkedHashSet<String>(8);
      this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);
    }
    dependenciesForBean.add(canonicalName);
  }
}
````

* 根据对象的作用域创建相应的对象

在确保依赖的对象都已经创建完成后，就可以开始创建当前对象了. 这里要根据对象的作用域(scope)来采用不同的方式创建. 先来看看singleton对象的创建. 这里调用了getSingleton方法来获取对象，并提供了匿名的ObjectFactory实现. 在getSingleton方法中，其实就是回调了ObjectFactory类的getObject方法, 这是“策略模式”的实践. 在获取到sharedInstance对象后，会调用getObjectForBeanInstance方法，这个方法的作用在前面已经提及过，它是用来判断sharedInstance对象的类型，如果是FactoryBean，则调用相应的方法返回对象，如果它是普通对象或请求的就是工厂对象本身，则直接返回. 

关于createBean()方法的执行过程，下一篇会有详细的解读. 

````java
if (mbd.isSingleton()) {
  sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
      try {
        return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
        // Explicitly remove instance from singleton cache: It might have been put there
        // eagerly by the creation process, to allow for circular reference resolution.
        // Also remove any beans that received a temporary reference to the bean.
        destroySingleton(beanName);
        throw ex;
      }
    }
  });
  bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
````

再来看看prototype对象的创建. 由于prototype域对象的特性是每次请求对象时都返回新的对象，因此这里的创建过程相对简单，就是调用createBean()方法返回新的实例，再调用getObjectForBeanInstance()方法进行转换即可. 

````java
else if (mbd.isPrototype()) {
  // It's a prototype -> create a new instance.
  Object prototypeInstance = null;
  try {
    beforePrototypeCreation(beanName);
    prototypeInstance = createBean(beanName, mbd, args);
  }
  finally {
    afterPrototypeCreation(beanName);
  }
  bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
````

除此之外，Spring还支持自定义Scope，对于自定义Scope的对象来讲，它的创建过程与之前的两种情况大同小异，这里就不做过多阐述.

````java
else {
  String scopeName = mbd.getScope();
  final Scope scope = this.scopes.get(scopeName);
  if (scope == null) {
    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
  }
  try {
    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
      @Override
      public Object getObject() throws BeansException {
        beforePrototypeCreation(beanName);
        try {
          return createBean(beanName, mbd, args);
        }
        finally {
          afterPrototypeCreation(beanName);
        }
      }
    });
    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
  }
  catch (IllegalStateException ex) {
    throw new BeanCreationException(beanName,
                                    "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                    "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                    ex);
  }
}
````

#### 2.3.2.4 类型转换

在经过上述步骤后，bean对象的实例就基本上创建好了，这时候还需要再进一步判断当前获取的实例对象类型与实际请求的类型是否匹配，如果不匹配的话，还需要进行类型转换操作. 类型转换主要是通过TypeConverter接口来完成的. 

````java
// Check if required type matches the type of the actual bean instance.
if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
  try {
    return getTypeConverter().convertIfNecessary(bean, requiredType);
  }
  catch (TypeMismatchException ex) {
    if (logger.isDebugEnabled()) {
      logger.debug("Failed to convert bean '" + name + "' to required type [" +
                   ClassUtils.getQualifiedName(requiredType) + "]", ex);
    }
    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
  }
}
````

