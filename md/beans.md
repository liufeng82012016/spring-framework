# Beans 学习
### 核心类
1. DefaultListableBeanFactory
是整个Bean加载的核心部分，Spring注册及加载Bean的默认实现，综合父类功能，主要是对bean注册后的处理
![DefaultListableBeanFactory类图](./img/XmlBeanFactory-UML.png)
2. AliasRegistry:定义对Alias的简单增删改操作
3. SimpleAliasRegistry：主要使用map作为alias的缓存，并对接口AliasRegistry进行实现
4. SingletonBeanRegistry：定义对单例的注册及获取
5. BeanFactory：定义获取bean及bean的各种属性
6. DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry各函数实现
7. HierarchicalBeanFactory：继承BeanFactory，增加对ParentFactory的支持
8. BeanDefinitionRegistry：定义对BeanDefinition的增删改操作
9. FactoryBeanRegistrySupport：在DefaultSingletonBeanRegistry增加对FactoryBean的特殊处理
10. ConfigurationBeanFactory：提供配置Factory的各种方法
11. ListableBeanFactory：根据各种条件获取bean的配置清单
12. AbstractBeanFactory：综合FactoryBeanRegistrySupport和ConfigurationBeanFactory的
13. AutowireCapableBeanFactory：提供创建Bean、自动注入、初始化以及应用后处理器
14. AbstractAutowireCapableBeanFactory：综合AbstractBeanFactory，并对AutowireCapableBeanFactory接口进行实现
15. ConfigurationListableBeanFactory：BeanFactory配置清单，指定忽略类型和接口等
16. XmlBeanFactory：对DefaultListableBeanFactory进行扩展，主要用于从XML文档读取BeanDefinition，主要变化在于定义XmlBeanDefinitionReader的reader属性
### XmlBeanFactory 
1. ResourceLoader：定义资源加载器，主要应用于根据给定的资源文件地址返回对应的Resource
    1.1 Java中将不同来源的资源抽象成URL，通过注册不同的handler（URLStreamHandler）来处理
    1.2 Spring使用Resource实现，如URLResource、ClassPathResource
2. BeanDefinitionReader：主要定义资源文件读取并转换为BeanDefinition
3. EnvironmentCapable：定义获取Environment的方法
4. DocumentLoader：定义从资源文件加载到转换为Document的方法
5. AbstractBeanDefinitionReader：对EnvironmentCapable、BeanDefinitionReader进行实现
6. BeanDefinitionDocumentReader：定义读取BeanDefinition并注册BeanDefinition的功能
7. BeanDefinitionParserDelegate：定义解析Element的方法


### getBean全流程（singleton模式）org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean
```java
class  AbstractBeanFactory{
protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
	    // 转换beanName，如别名
		String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		// 检查缓存或实例工厂是否有对应的实例
		// 因为创建单例bean的时候可能存在依赖注入，这里是为了避免循环依赖。提前将实例或factory暴露出来，可以直接引用
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// sharedInstance可能是BeanFactory，
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			// 如果当前BeanFactory不存在对应BeanDefinition，且存在父工厂
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				// 但是，如果parent也没有，怎么处理的？不存在这种情况？
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
			// 不仅仅是类型检查，标记Bean正在创建
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				// 将BeanDefinition转换为RootBeanDefinition，如果是子Bean，合并父类的属性
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					// 遍历所有的依赖，递归实例化所有依赖的bean
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						} catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					// 真正创建单例
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						} catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					// 如果是Factory，要获取真正的实例对象返回
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				} else if (mbd.isPrototype()) {
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
				} else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					} catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			} catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
}
```
1. 解析beanName，因为传入的参数可能是别名，也可能是BeanFactory
2. 尝试获取单例 org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(java.lang.String, boolean)
```java
public class DefaultSingletonBeanRegistry{
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	// 首先尝试从实例化完成的map中获取对象
		Object singletonObject = this.singletonObjects.get(beanName);
		// 如果为空，且单例正在构建
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				// 从提前曝光的单例map中获取
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					// 如果还没有提前曝光，且允许提前曝光，就会尝试获取BeanFactory
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						// 返回BeanFactory
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
					// 问题，如果不允许提前曝光，BeanFactory可能会重复创建吗？map去重？
				}
			}
		}
		return singletonObject;
	}
}
```
3. Bean实例化 org.springframework.beans.factory.support.AbstractBeanFactory.getObjectForBeanInstance
```java
class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry{
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		// 锁住整个map
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				// 如果为空，且正在创建中
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				// 前置处理，默认实现是记录加载状态
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 初始化Bean
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				} catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				} catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					// 后置处理，默认实现是移除加载状态
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					// 如果是新的单例，加入到map（4级）
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}	
}
class  AbstractBeanFactory  extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory{
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		// FactoryBean的bean名称已&开头
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
		    // 如果不是FactoryBean，是实例，直接返回
		    // 如果是FactoryBean，且调用者需要返回的正式FactoryBean，才返回
			return beanInstance;
		}
        // 利用FactoryBean真正创建实例
		Object object = null;
		if (mbd == null) {
			// 尝试从缓存加载对象
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			// 缓存中无该对象，创建对象
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			// 如果BeanDefinition参数是空，但实际上注册过
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			// synthetic为true，表示是应用程序本身定义的，而非用户定义
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
	
	

}

class FactoryBeanRegistrySupport {
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        		if (factory.isSingleton() && containsSingleton(beanName)) {
        			synchronized (getSingletonMutex()) {
        				// 如果是单例，加锁，再尝试从缓存获取
        				Object object = this.factoryBeanObjectCache.get(beanName);
        				if (object == null) {
        					object = doGetObjectFromFactoryBean(factory, beanName);
        					// Only post-process and store if not put there already during getObject() call above
        					// (e.g. because of circular reference processing triggered by custom getBean calls)
        					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
        					if (alreadyThere != null) {
        						object = alreadyThere;
        					} else {
        						// 需要后置处理（只有调用了getObject()方法才需要后置处理）
        						if (shouldPostProcess) {
        							if (isSingletonCurrentlyInCreation(beanName)) {
        								// Temporarily return non-post-processed object, not storing it yet..
        								return object;
        							}
        							beforeSingletonCreation(beanName);
        							try {
        								// AbstractBeanFactory内什么也没做，子类AbstractAutowireCapableBeanFactory有默认实现
        								object = postProcessObjectFromFactoryBean(object, beanName);
        							}
        							catch (Throwable ex) {
        								throw new BeanCreationException(beanName,
        										"Post-processing of FactoryBean's singleton object failed", ex);
        							}
        							finally {
        								afterSingletonCreation(beanName);
        							}
        						}
        						if (containsSingleton(beanName)) {
        							this.factoryBeanObjectCache.put(beanName, object);
        						}
        					}
        				}
        				return object;
        			}
        		} else {
        			Object object = doGetObjectFromFactoryBean(factory, beanName);
        			if (shouldPostProcess) {
        				try {
        					object = postProcessObjectFromFactoryBean(object, beanName);
        				}
        				catch (Throwable ex) {
        					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
        				}
        			}
        			return object;
        		}
        	}
        	
        	
        	private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
            		Object object;
            		try {
            			// 需要权限验证
            			if (System.getSecurityManager() != null) {
            				AccessControlContext acc = getAccessControlContext();
            				try {
            					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
            				} catch (PrivilegedActionException pae) {
            					throw pae.getException();
            				}
            			}
            			else {
            				// 获取bean
            				object = factory.getObject();
            			}
            		} catch (FactoryBeanNotInitializedException ex) {
            			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
            		} catch (Throwable ex) {
            			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
            		}
            
            		// Do not accept a null value for a FactoryBean that's not fully
            		// initialized yet: Many FactoryBeans just return null then.
            		if (object == null) {
            			// bean正在创建，外层不是加锁了吗？
            			if (isSingletonCurrentlyInCreation(beanName)) {
            				throw new BeanCurrentlyInCreationException(
            						beanName, "FactoryBean which is currently in creation returned null from getObject");
            			}
            			object = new NullBean();
            		}
            		return object;
            	}
}

class AbstractAutowireCapableBeanFactory{
	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
    		return applyBeanPostProcessorsAfterInitialization(object, beanName);
	}
    		@Override
        	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
        			throws BeansException {
        
        		Object result = existingBean;
        		// 调用后置处理器的postProcessAfterInitialization方法
        		for (BeanPostProcessor processor : getBeanPostProcessors()) {
        			Object current = processor.postProcessAfterInitialization(result, beanName);
        			if (current == null) {
        				return result;
        			}
        			result = current;
        		}
        		return result;
        	}
        	
    @Override
 	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
 			throws BeanCreationException {
 
 		if (logger.isDebugEnabled()) {
 			logger.debug("Creating instance of bean '" + beanName + "'");
 		}
 		RootBeanDefinition mbdToUse = mbd;
 
 		// Make sure bean class is actually resolved at this point, and
 		// clone the bean definition in case of a dynamically resolved Class
 		// which cannot be stored in the shared merged bean definition.
 		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
 		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
 			mbdToUse = new RootBeanDefinition(mbd);
 			mbdToUse.setBeanClass(resolvedClass);
 		}
 
 		// Prepare method overrides.验证及准备要复写的方法，准备过程在AbstractBeanDefinition类
 		try {
 			mbdToUse.prepareMethodOverrides();
 		} catch (BeanDefinitionValidationException ex) {
 			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
 					beanName, "Validation of method overrides failed", ex);
 		}
 
 		try {
 			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
 			// 给BeanPostProcessors一个机会返回一个代理类，替代真正的实例。这一行代码的关键作用在于是否做AOP增强
 			// 注意，如果在这里生成了Bean，直接返回，不进行doCreateBean过程
 			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
 			if (bean != null) {
 				return bean;
 			}
 		} catch (Throwable ex) {
 			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
 					"BeanPostProcessor before instantiation of bean failed", ex);
 		}
 
 		try {
 			// 真正创建实例
 			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
 			if (logger.isDebugEnabled()) {
 				logger.debug("Finished creating instance of bean '" + beanName + "'");
 			}
 			return beanInstance;
 		} catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
 			// A previously detected exception with proper bean creation context already,
 			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
 			throw ex;
 		} catch (Throwable ex) {
 			throw new BeanCreationException(
 					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
 		}
 	}
 
 	@Nullable
 	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
 		Object bean = null;
 		// 如果没有被解析
 		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
 			// Make sure bean class is actually resolved at this point.
 			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
 				Class<?> targetType = determineTargetType(beanName, mbd);
 				if (targetType != null) {
 					// 后置处理
 					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
 					if (bean != null) {
 						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
 					}
 				}
 			}
 			mbd.beforeInstantiationResolved = (bean != null);
 		}
 		return bean;
 	}
 	
    @Nullable
 	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
 		for (BeanPostProcessor bp : getBeanPostProcessors()) {
 			if (bp instanceof InstantiationAwareBeanPostProcessor) {
 				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
 				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
 				if (result != null) {
 					return result;
 				}
 			}
 		}
 		return null;
 	}
 	
    @Override
 	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
 			throws BeansException {
 
 		Object result = existingBean;
 		for (BeanPostProcessor processor : getBeanPostProcessors()) {
 			Object current = processor.postProcessAfterInitialization(result, beanName);
 			if (current == null) {
 				return result;
 			}
 			result = current;
 		}
 		return result;
 	}
 	
    protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
 			throws BeanCreationException {
 
 		// Instantiate the bean.
 		BeanWrapper instanceWrapper = null;
 		if (mbd.isSingleton()) {
 			// 从缓存获取
 			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
 		}
 		if (instanceWrapper == null) {
 			// 缓存获取为空，创建实例
 			instanceWrapper = createBeanInstance(beanName, mbd, args);
 		}
 		Object bean = instanceWrapper.getWrappedInstance();
 		Class<?> beanType = instanceWrapper.getWrappedClass();
 		if (beanType != NullBean.class) {
 			mbd.resolvedTargetType = beanType;
 		}
 
 		// Allow post-processors to modify the merged bean definition.
 		synchronized (mbd.postProcessingLock) {
 			if (!mbd.postProcessed) {
 				try {
 					// 应用MergedBeanDefinitionPostProcessors
 					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
 				}
 				catch (Throwable ex) {
 					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
 							"Post-processing of merged bean definition failed", ex);
 				}
 				mbd.postProcessed = true;
 			}
 		}
 
 		// Eagerly cache singletons to be able to resolve circular references
 		// even when triggered by lifecycle interfaces like BeanFactoryAware.
 		// 是否允许提早曝光：单例&&允许循环依赖&&当前bean正在创建中
 		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
 				isSingletonCurrentlyInCreation(beanName));
 		if (earlySingletonExposure) {
 			if (logger.isDebugEnabled()) {
 				logger.debug("Eagerly caching bean '" + beanName +
 						"' to allow for resolving potential circular references");
 			}
 			// 为避免后期循环依赖，初始化完成前将ObjectFactory加入工厂
 			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
 			/**
 			* 应用SmartInstantiationAwareBeanPostProcessor
 			* protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
            * 		Object exposedObject = bean;
            *  		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            * 			for (BeanPostProcessor bp : getBeanPostProcessors()) {
            *  				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
            * 					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
            * 					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            * 				}
            * 			}
            * 		}
            *  		return exposedObject;
            *  	}
            */
 		}
 
 		// Initialize the bean instance.
 		Object exposedObject = bean;
 		try {
 			// 初始化Bean，并对属性填充；可能会递归初始化依赖Bean
 			populateBean(beanName, mbd, instanceWrapper);
 			// 调用初始化方法，如init-method
 			exposedObject = initializeBean(beanName, exposedObject, mbd);
 		}
 		catch (Throwable ex) {
 			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
 				throw (BeanCreationException) ex;
 			}
 			else {
 				throw new BeanCreationException(
 						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
 			}
 		}
 
 		if (earlySingletonExposure) {
 			Object earlySingletonReference = getSingleton(beanName, false);
 			// earlySingletonReference 仅在检测出循环依赖时不为空
 			if (earlySingletonReference != null) {
 				if (exposedObject == bean) {
 					// exposedObject在初始方法没有被改变，也就是没有增强
 					exposedObject = earlySingletonReference;
 				} else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
 					String[] dependentBeans = getDependentBeans(beanName);
 					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
 					for (String dependentBean : dependentBeans) {
 						// 检测依赖
 						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
 							actualDependentBeans.add(dependentBean);
 						}
 					}
 					// 
 					if (!actualDependentBeans.isEmpty()) {
 						throw new BeanCurrentlyInCreationException(beanName,
 								"Bean with name '" + beanName + "' has been injected into other beans [" +
 								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
 								"] in its raw version as part of a circular reference, but has eventually been " +
 								"wrapped. This means that said other beans do not use the final version of the " +
 								"bean. This is often the result of over-eager type matching - consider using " +
 								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
 					}
 				}
 			}
 		}
 
 		// Register bean as disposable.
 		try {
 			// 根据score注册bean，如果配置了destory-method，这里需要注册以便于在销毁的时候调用
 			registerDisposableBeanIfNecessary(beanName, bean, mbd);
 		}
 		catch (BeanDefinitionValidationException ex) {
 			throw new BeanCreationException(
 					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
 		}
 
 		return exposedObject;
 	}

}

class AbstractBeanDefinition  extends BeanMetadataAttributeAccessor
      		implements BeanDefinition, Cloneable {
	
		public void prepareMethodOverrides() throws BeanDefinitionValidationException {
    		// Check that lookup methods exist and determine their overloaded status.
    		if (hasMethodOverrides()) {
    			getMethodOverrides().getOverrides().forEach(this::prepareMethodOverride);
    		}
    	}

	protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
	        // 方法前置处理，根据Bean名称和MethodName获取对应的方法
    		int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
    		if (count == 0) {
    			throw new BeanDefinitionValidationException(
    					"Invalid method override: no method with name '" + mo.getMethodName() +
    					"' on class [" + getBeanClassName() + "]");
    		}
    		else if (count == 1) {
    			// Mark override as not overloaded, to avoid the overhead of arg type checking.
    			// 标记override，避免参数类型检查
    			mo.setOverloaded(false);
    		}
    		// 如果有多个重载方法，这里不做处理
    	}
}
```
### FactoryBean 
1. 工厂接口，当class是此接口的子类时，getBean()获取的不是FactoryBean本身，而是getObject()返回的对象

### 循环依赖
1. 构造器循环依赖，无法解决
2. Setter循环依赖，可通过Spring容器提前暴露刚完成构造器但未完成其他步骤的bean来完成，只能解决单例模式下的循环依赖
3. prototype循环依赖，无法解决

### 容器扩展功能
1. BeanFactory和ApplicationContext都是用于加载bean，但后者提供了更多的扩展功能，除非在一些限制场合如字节长度对内容有很大影响时，通常选用后者
2. 区别




