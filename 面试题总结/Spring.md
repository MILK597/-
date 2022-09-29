# Bean的生命周期

1. 实例化

2. 属性注入

3. 如果Bean实现BeanName**Aware接口**执行setBeanName方法

   如果Bean实现BeanFactoryAware接口或者ApplicationContextAware接口，调用setBeanFactory或setApplicationContext方法。

4. 如果Bean实现了**BeanPostProcessor接口**，调用**postProcessBeforeInitialization方法**

5. 如果Bean实现**InitializingBean接口**执行
   afterPropertiesSet方法

6. 如果在xml配置文件中指定了**init-method方法**，执行它

7. 如果Bean实现了**BeanPostProcessor接口**，执行**postProcessAfterInitialization方法**

8. 注册必要的Destruction相关回调接口

   ​	为了方便对象的销毁，在此处调用注销的回调接口，方便对象进行销毁操作

9. 使用Bean

10. 如果Bean实现**DisposableBean**执行
   destroy方法

11. 如果在xml配置文件中指定了**destroy-method方法**，执行它

# Bean注入属性有哪几种方式

spring支持 构造器注入 和 setter方法注入