# mybatis源码学习

## 前言
学而时习之，不亦说乎？
先明确学习常用优秀框架如Spring、Mybatis等源码的目的：
* 初级目标：了解源码的架构设计、代码设计、及运行原理。
* 中级目标：能够对其进行扩展，比如实现自定义插件等。
* 高级目标：能够发现源码中的bug或能为后续版本进行开发。

一般而言，至少要达到初级目标才会有意义，能够达到中级目标就很不错了，能达到高级目标的就很难得了。


## Mybatis运作流程
<图>

## Mybatis Plugin
### Mybatis插件实现原理
Mybatis插件的作用是在SQL执行的各阶段能够自定义做一些操作，比如打印日志、分页、乐观锁更新判断等，思想与AOP有类似之处，具体实现方式是使用JDK动态代理。  
有几个定义：
* 目标执行对象：Object target，可以是StatementHandler、ParameterHandler、Executor、ResultSetHandler
* 插件实现对象：Interceptor interceptor，需要实现Interceptor接口

**Interceptor接口**
```java
public interface Interceptor {

  /**
   * 调用插件对象方法时的执行入口
   * @see org.apache.ibatis.plugin.Plugin#invoke(Object, Method, Object[])
   * 如果符合插件定义的方法（通过@Signature判断），则会执行插件的intercept
   * @param invocation
   * @return
   * @throws Throwable
   */
  Object intercept(Invocation invocation) throws Throwable;

  /**
   * 该方法主要作用
   * 1.可以根据target的Class类型来判断是否生成插件对象（代理对象） -- 此步骤可无，因为在Plugin.wrap也会根据Signature进行判断
   * 2.使用Plugin.wrap(Object target, Interceptor interceptor)生成代理对象
   * @param target:宿主对象，比如Executor
   * @return
   */
  Object plugin(Object target);

  /**
   * 设置相关属性
   * @param properties
   */
  void setProperties(Properties properties);

}
```

**Plugin#wrap**
由目标对象target生成代理对象的关键方法为Plugin#wrap，下面看下相关代码：  
```
  /**
   *
   * @param target：目标对象，比如Executor
   * @param interceptor
   * @return
   */
  public static Object wrap(Object target, Interceptor interceptor) {
    // 抽取插件实现类上@Intercepts注解内容，Map的key为@Signature注解的type，Map的value为@Signature注解定义method(args..)对应的Method
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    // 筛选出宿主对象实现的所有接口中在@Intercepts注解中配置的@Signature的type
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    // 只有存在@Signature的type为目标对象的class时才会生成代理对象
    if (interfaces.length > 0) {
      // 生成代理对象
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
```
由JDK动态代理的实现调用原理可知，Plugin实现了InvocationHandler接口，生成代理对象后执行代理对象的目标方法时会调用new Plugin(target, interceptor, signatureMap)#invoke方法，代码如下：  
```java
public class Plugin implements InvocationHandler {  
  // ...

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      // 先判断执行的方法对应类是否在signatureMap中存在
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        // 执行插件的intercept()方法，此方法中会调用target的方法，且会做插件自定义的事情
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
}
```


### 插件设计思想

### 自定义插件实现


