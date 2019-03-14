## Tomcat的生命周期
* Tomcat中的容器生命周期实现。
> 所有容器的转态转换（如新建、初始化、启动、停止等）都是由外到内，由上到下进行，即先执行父容器的状态转换及相关操作，然后再执行子容器的转态转换，这个过程是层层迭代执行的。
>> 容器的新建
>>> 所有容器在构造的过程中，都会首先对父类LifecycleBase进行构造。LifecycleBase中定义了所有容器的起始状态为LifecycleState.NEW
####
    private volatile LifecycleState state = LifecycleState.NEW;    
    
> >容器的初始化
> > >每个容器的init方法是自身初始化的入口
####
    调用方-->init()-->LifecycleBase-->initInternal()-->具体容器-->initInternal()-->
    LifecycleMBeanBase-->regist()-->LifecycleSupport-->调用子容器的init()-->
    LifecycleState.INITIALIZED()-->调用方
    备：所说的具体容器，实际就是LifecycleBase的具体实现类
>>步骤
>>> - 1. 调用方调用容器父类LifecycleBase的init方法，LifecycleBase的init方法主要完成一些所有容器公共抽象出来的动作；
>>> - 2. LifecycleBase的init方法调用具体容器的initInternal方法实现，此initInternal方法用于对容器本身真正的初始化；
>>> - 3. 具体容器的initInternal方法调用父类LifecycleMBeanBase的initInternal方法实现，此initInternal方法用于将容器托管到JMX，便于运维管理；
>>> - 4. LifecycleMBeanBase的initInternal方法调用自身的register方法，将容器作为MBean注册到MBeanServer；
>>> - 5. 容器如果有子容器，会调用子容器的init方法；
>>> - 6. 容器初始化完毕，LifecycleBase会将容器的状态更改为初始化完毕，即LifecycleState.INITIALIZED。
####
    public synchronized final void init() throws LifecycleException {
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.INIT_EVENT);
        }
 
        initInternal();
        
        setState(LifecycleState.INITIALIZED);
    }
    以上：只有当前容器的状态处于LifecycleState.NEW的才可以被初始化，真正执行初始化的方法是initInternal，当初始化完毕，当前容器的状态会被更改为LifecycleState.INITIALIZED。
    为了简便起见，我们还是以StandardServer这个容器为例，StandardServer的initInternal方法的实现
####
    @Override
    protected void initInternal() throws LifecycleException {
        
        super.initInternal();
 
        // Register global String cache geng
        // Note although the cache is global, if there are multiple Servers
        // present in the JVM (may happen when embedding) then the same cache 
        // will be registered under multiple names
        onameStringCache = register(new StringCache(), "type=StringCache");
 
        // Register the MBeanFactory
        onameMBeanFactory = register(new MBeanFactory(), "type=MBeanFactory");
        
        // Register the naming resources
        onameNamingResoucres = register(globalNamingResources,
                "type=NamingResources");
        
        // Initialize our defined Services
        for (int i = 0; i < services.length; i++) {
            services[i].init();
        }
    }
>> 将当前容器注册到JMX
####
    @Override
    protected void initInternal() throws LifecycleException {
        
        // If oname is not null then registration has already happened via jiaan
        // preRegister().
        if (oname == null) {
            mserver = Registry.getRegistry(null, null).getMBeanServer();
            
            oname = register(this, getObjectNameKeyProperties());
        }
    }
    以上：调用父类LifecycleBase的initInternal方法，为当前容器创建DynamicMBean，并注册到JMX中。

    @Override
    protected final String getObjectNameKeyProperties() {
        return "type=Server";
    }
    以上：StandardServer实现的getObjectNameKeyProperties方法如上
    
    protected final ObjectName register(Object obj,
            String objectNameKeyProperties) {
        
        // Construct an object name with the right domain
        StringBuilder name = new StringBuilder(getDomain());
        name.append(':');
        name.append(objectNameKeyProperties);
 
        ObjectName on = null;
 
        try {
            on = new ObjectName(name.toString());
            
            Registry.getRegistry(null, null).registerComponent(obj, on, null);
        } catch (MalformedObjectNameException e) {
            log.warn(sm.getString("lifecycleMBeanBase.registerFail", obj, name),
                    e);
        } catch (Exception e) {
            log.warn(sm.getString("lifecycleMBeanBase.registerFail", obj, name),
                    e);
        }
 
        return on;
    }
    以上：LifecycleBase的register方法（见代码清单9）会为当前容器创建对应的注册名称，以StandardServer为例，
    getDomain默认返回Catalina，因此StandardServer的JMX注册名称默认为Catalina:type=Server，
    真正的注册在registerComponent方法中实现。
    
    /** Register a component 
     * XXX make it private 
     * 
     * @param bean
     * @param oname
     * @param type
     * @throws Exception
     */ 
    public void registerComponent(Object bean, ObjectName oname, String type)
           throws Exception
    {
        if( log.isDebugEnabled() ) {
            log.debug( "Managed= "+ oname);
        }
 
        if( bean ==null ) {
            log.error("Null component " + oname );
            return;
        }
 
        try {
            if( type==null ) {
                type=bean.getClass().getName();
            }
 
            ManagedBean managed = findManagedBean(bean.getClass(), type);
 
            // The real mbean is created and registered
            DynamicMBean mbean = managed.createMBean(bean);
 
            if(  getMBeanServer().isRegistered( oname )) {
                if( log.isDebugEnabled()) {
                    log.debug("Unregistering existing component " + oname );
                }
                getMBeanServer().unregisterMBean( oname );
            }
 
            getMBeanServer().registerMBean( mbean, oname);
        } catch( Exception ex) {
            log.error("Error registering " + oname, ex );
            throw ex;
        }
    }
    以上：Registry的registerComponent方法会为当前容器（如StandardServer）创建DynamicMBean，并且注册到MBeanServer。
    
>> 步骤二 
>>> 将StringCache、MBeanFactory、globalNamingResources注册到JMX
####
    StringCache的注册名为Catalina:type=StringCache，
    MBeanFactory的注册名为Catalina:type=MBeanFactory，
    globalNamingResources的注册名为Catalina:type=NamingResources。

>> 步骤三
>>>初始化子容器
####
    从代码清单7中看到StandardServer主要对Service子容器进行初始化，默认是StandardService。
    注意：个别容器并不完全遵循以上的初始化过程.
    比如ProtocolHandler作为Connector的子容器，其初始化过程并不是由Connector的initInternal方法调用的，而是与启动过程一道被Connector的startInternal方法所调用。
    
>>容器启动
####
    调用方-->start()-->LifecycleBase-->LifecycleState.STARTING_PREP()-->具体容器-->
    startInternal()-->具体容器-->LifecycleState.STARTING()-->具体容器-->调用子容器的start()-->
    具体容器-->LifecycleState.STARTED()--LifecycleBase--调用方
>>步骤如下
>>> - 1. 调用方调用容器父类LifecycleBase的start方法，LifecycleBase的start方法主要完成一些所有容器公共抽象出来的动作；
>>> - 2. LifecycleBase的start方法先将容器状态改为LifecycleState.STARTING_PREP，然后调用具体容器的startInternal方法实现，此startInternal方法用于对容器本身真正的初始化；
>>> - 3. 具体容器的startInternal方法会将容器状态改为LifecycleState.STARTING，容器如果有子容器，会调用子容器的start方法启动子容器；
>>> - 4. 容器启动完毕，LifecycleBase会将容器的状态更改为启动完毕，即LifecycleState.STARTED。
####
        @Override
    public synchronized final void start() throws LifecycleException {
        
        if (LifecycleState.STARTING_PREP.equals(state) ||
                LifecycleState.STARTING.equals(state) ||
                LifecycleState.STARTED.equals(state)) {
            
            if (log.isDebugEnabled()) {
                Exception e = new LifecycleException();
                log.debug(sm.getString("lifecycleBase.alreadyStarted",
                        toString()), e);
            } else if (log.isInfoEnabled()) {
                log.info(sm.getString("lifecycleBase.alreadyStarted",
                        toString()));
            }
            
            return;
        }
        
        if (state.equals(LifecycleState.NEW)) {
            init();
        } else if (!state.equals(LifecycleState.INITIALIZED) &&
                !state.equals(LifecycleState.STOPPED)) {
            invalidTransition(Lifecycle.BEFORE_START_EVENT);
        }
 
        setState(LifecycleState.STARTING_PREP);
 
        try {
            startInternal();
        } catch (LifecycleException e) {
            setState(LifecycleState.FAILED);
            throw e;
        }
 
        if (state.equals(LifecycleState.FAILED) ||
                state.equals(LifecycleState.MUST_STOP)) {
            stop();
        } else {
            // Shouldn't be necessary but acts as a check that sub-classes are
            // doing what they are supposed to.
            if (!state.equals(LifecycleState.STARTING)) {
                invalidTransition(Lifecycle.AFTER_START_EVENT);
            }
            
            setState(LifecycleState.STARTED);
        }
    }
    以上：说明在真正启动容器之前需要做2种检查：
    1.如果当前容器已经处于启动过程（即容器状态为LifecycleState.STARTING_PREP、LifecycleState.STARTING、
    LifecycleState.STARTED）中，则会产生并且用日志记录LifecycleException异常并退出。
    2.如果容器依然处于LifecycleState.NEW状态，则在启动之前，首先确保初始化完毕。
    
    还说明启动容器完毕后，需要做1种检查。
    即如果容器启动异常导致容器进入LifecycleState.FAILED或者LifecycleState.MUST_STOP状态，则需要调用stop方法停止容器。
>>现在我们重点分析startInternal方法，还是以StandardServer为例
####
    @Override
    protected void startInternal() throws LifecycleException {
 
        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
        setState(LifecycleState.STARTING);
 
 
        // Start our defined Services
        synchronized (services) {
            for (int i = 0; i < services.length; i++) {
                services[i].start();
            }
        }
    }
    以上：StandardServer的启动由以下步骤组成：
    1.产生CONFIGURE_START_EVENT事件；
    2.将自身状态更改为LifecycleState.STARTING；
    3.调用子容器Service（默认为StandardService）的start方法启动子容器。
    
    备：除了初始化、启动外，各个容器还有停止和销毁的生命周期，其原理与初始化、启动类似
>> Tomcat启动完毕后，打开Java visualVM，打开Tomcat进程监控，给visualVM安装MBeans插件后，选择MBeans标签页可以对Tomcat所有注册到JMX中的对象进行管理，比如StandardService就向JMX暴露了start和stop等方法，这样管理员就可以动态管理Tomcat。

>> 总结
>>> Tomcat通过将内部所有组件都抽象为容器，为容器提供统一的生命周期管理，各个子容器只需要关心各自的具体实现，这便于Tomcat以后扩展更多的容器，对于研究或者学习Tomcat的人来说，其设计清晰易懂。

