# 一文理清Dubbo关键注解@Adaptive
[TOC]
## 0.前言
### 说明
本文Dubbo源码版本为2.7.8-release。
  
理解Dubbo SPI对阅读Dubbo源码很重要，而最关键的类是ExtensionLoader，本文会对与@Adaptive有关的涉及ExtensionLoader部分进行说明，ExtensionLoader类其余部分不会展开。
阅读此文的前提是对Dubbo SPI以及Dubbo的ExtensionLoader类有所了解。@Adaptive是辅助ExtentionLoader使用的一部分。

### 几个问题
带着下面几个问题去学习@Adaptive：
* 1.@Adaptive注解有什么作用，适用于什么场景，实现的原理是怎样？
* 2.@Adaptive修饰class与method有什么区别？
* 3.针对一个Dubbo SPI接口，使用@Adaptive修饰两个及以上实现类时，调用ExtensionLoader.getAdaptiveExtension会发生什么？

## 1.@Adaptive概览
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```
从Adaptive注解定义及源码注释中可以得出以下几点：  
* Adaptive是一个注解，可以修饰类（接口，枚举）和方法
* 此注解的作用是为ExtensionLoader注入扩展实例提供有用的信息
  
从注释中理解value的作用：  
* 1.配置value用于决定选择使用具体的扩展类，默认为空string数组。
* 2.通过value配置的值key，在修饰的方法的入参org.apache.dubbo.common.URL中通过key获取到对应的值value，根据value作为extensionName去决定使用对应的扩展类。
* 3.如果通过2没有找到对应的扩展（extName->ExtensionClass），会选择默认的扩展类，即@SPI配置默认扩展类

> tips：为什么要从方法的入参org.apache.dubbo.common.URL获取值？

### 举个例子
从一个例子来大致了解下@Adaptive的作用及使用方法，例子来源Dubbo单元测试部分代码。  
```java
/**
* Dubbo SPI 接口
*/
@SPI("impl1")
public interface SimpleExt {
    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);
}
```
如果调用ExtensionLoader.getExtensionLoader(SimpleExt.class).getAdaptiveExtension().yell(url, s)方法，最终调用哪一个扩展类实例的流程为：    
首先，通过url.getParameters.get("key1")获取，没有获取到则用url.getParameters.get("key2"),如果还是没有获取到则使用impl1对应的实现类，最后还是没有获取到则抛异常IllegalStateException。
  
可以看出，@Adaptive的好处就是可以通过方法入参决定具体调用哪一个实现类。下面会对@Adaptive的具体实现进行详细分析。
  
## 2.实现流程
获取自适应扩展类的使用方法，通常使用ExtensionLoader.getExtensionLoader(Xxxx.class).getAdaptiveExtension()来获取Adaptive实例，如下，引用自ServiceConfig获取PROTOCOL的方式。
> private static final Protocol PROTOCOL = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

### 获取Adaptive实例的流程
<流程图>

流程关键点说明：  
1.上图中黄色标记的，cachedAdaptiveClass是在ExtensionLoader#loadClass方法中加载Extension类时缓存的。  
2.上图中绿色标记的，如果Extension类中存在被@Adaptive修饰的类时会使用该类来初始化实例。    
3.上图中红色标记的，如果Extension类中不存在被@Adaptive修饰的类时，则需要动态生成代码，通过javassist（默认）来编译生成Xxxx$Adaptive类来实例化。  
4.实例化后通过injectExtension来将Adaptive实例的Extension注入（属性注入）。  
  
后续围绕上述的关键点3详细展开，关键点4后续完善。
  
## 3.动态生成Adaptive类
下面的代码为动态生成Adaptive类的相关代码，具体生成代码的细节在AdaptiveClassCodeGenerator#generate中
```java
public class ExtensionLoader<T> {
    // ...

    private Class<?> getAdaptiveExtensionClass() {
        // 根据对应的SPI文件加载扩展类并缓存，细节此处不展开
        getExtensionClasses();
        // 如果存在被@Adaptive修饰的类则直接返回此类
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        // 动态生成Xxxx$Adaptive类
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }

    private Class<?> createAdaptiveExtensionClass() {
        // 生成Xxxx$Adaptive类代码，可自行加日志或断点查看生成的代码
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        ClassLoader classLoader = findClassLoader();
        // 获取动态编译器，默认为javassist
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
}
```
AdaptiveClassCodeGenerator#generate生成code的方式是通过字符串拼接，大量使用String.format,整个代码过程比较繁琐，可通过debug去了解细节。  
最关键的的部分是生成被@Adaptive修饰的方法的内容，也就是最终调用实例的@Adaptive方法时，可通过参数来动态选择具体使用哪个扩展实例。下面对此部分进行分析：  
```java
public class AdaptiveClassCodeGenerator {
    // ...
    /**
     * generate method content
     */
    private String generateMethodContent(Method method) {
        // 获取方法上的@Adaptive注解
        Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
        StringBuilder code = new StringBuilder(512);
        if (adaptiveAnnotation == null) {
            // 方法时没有@Adaptive注解，生成不支持的代码
            return generateUnsupported(method);
        } else {
            // 方法参数里URL是第几个参数，不存在则为-1
            int urlTypeIndex = getUrlTypeIndex(method);

            // found parameter in URL type
            if (urlTypeIndex != -1) {
                // Null Point check
                code.append(generateUrlNullCheck(urlTypeIndex));
            } else {
                // did not find parameter in URL type
                code.append(generateUrlAssignmentIndirectly(method));
            }
            // 获取方法上@Adaptive配置的value
            // 比如 @Adaptive({"key1","key2"}),则会返回String数组{"key1","key2"}
            // 如果@Adaptive没有配置value，则会根据简写接口名按驼峰用.分割，比如SimpleExt对应simple.ext
            String[] value = getMethodAdaptiveValue(adaptiveAnnotation);

            // 参数里是否存在org.apache.dubbo.rpc.Invocation
            boolean hasInvocation = hasInvocationArgument(method);

            code.append(generateInvocationArgumentNullCheck(method));
            // 生成String extName = xxx;的代码 ，extName用于获取具体的Extension实例
            code.append(generateExtNameAssignment(value, hasInvocation));
            // check extName == null?
            code.append(generateExtNameNullCheck(value));

            code.append(generateExtensionAssignment());

            // return statement
            code.append(generateReturnAndInvocation(method));
        }

        return code.toString();
    }

}
```  
上述生成Adaptive类的方法内容中最关键的步骤在生成extName的部分，也就是generateExtNameAssignment(value, hasInvocation)，此方法if太多了（有点眼花缭乱），此处对此方法的实现流程进行简单描述：  
_假设方法中的参数不包含org.apache.dubbo.rpc.Invocation，包好org.apache.dubbo.rpc.Invocation的情况后续再完善_  
(1)方法被@Adaptive修饰，没有配置value，且在接口@SPI上配置了默认的实现
```java
@SPI("impl1")
public interface SimpleExt {
    @Adaptive
    String echo(URL url, String s);
}
```
对应生成的代码为：  
> String extName = url.getParameter("simple.ext", "impl1")

(2)方法被@Adaptive修饰，没有配置value，且在接口@SPI上没有配置默认的实现
```java
@SPI
public interface SimpleExt {
    @Adaptive
    String echo(URL url, String s);
}
```
对应生成的代码为：  
> String extName = url.getParameter( "simple.ext")

(3)方法被@Adaptive修饰，配置了value（假设两个，依次类推），且在接口@SPI上配置了默认的实现
```java
@SPI("impl1")
public interface SimpleExt {
    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);
}
```
对应生成的代码为：  
> String extName = url.getParameter("key1", url.getParameter("key2", "impl1"));

(3)方法被@Adaptive修饰，配置了value（假设两个，依次类推），且在接口@SPI没有配置默认的实现
```java
@SPI
public interface SimpleExt {
    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);
}
```
对应生成的代码为：  
> String extName = url.getParameter("key1", url.getParameter("key2"));

### 完整案例
```java
@SPI("impl1")
public interface SimpleExt {
    // @Adaptive example, do not specify a explicit key.
    @Adaptive
    String echo(URL url, String s);

    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);

    // no @Adaptive
    String bang(URL url, int i);
}
```
生成的Adaptive类代码：
```java
package org.apache.dubbo.common.extension.ext1;
 
import org.apache.dubbo.common.extension.ExtensionLoader;
 
public class SimpleExt$Adaptive implements org.apache.dubbo.common.extension.ext1.SimpleExt {
 
    public java.lang.String yell(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("key1", url.getParameter("key2", "impl1"));
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.common.extension.ext1.SimpleExt) name from url (" + url.toString() + ") use keys([key1, key2])");
        org.apache.dubbo.common.extension.ext1.SimpleExt extension = (org.apache.dubbo.common.extension.ext1.SimpleExt)             ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.extension.ext1.SimpleExt.class).getExtension(extName);
        return extension.yell(arg0, arg1);
    }
 
    public java.lang.String echo(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("simple.ext", "impl1");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.common.extension.ext1.SimpleExt) name from url (" + url.toString() + ") use keys([simple.ext])");
        org.apache.dubbo.common.extension.ext1.SimpleExt extension = (org.apache.dubbo.common.extension.ext1.SimpleExt) ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.extension.ext1.SimpleExt.class).getExtension(extName);
        return extension.echo(arg0, arg1);
    }
 
    public java.lang.String bang(org.apache.dubbo.common.URL arg0, int arg1) {
        throw new UnsupportedOperationException("The method public abstract java.lang.String org.apache.dubbo.common.extension.ext1.SimpleExt.bang(org.apache.dubbo.common.URL,int) of interface org.apache.dubbo.common.extension.ext1.SimpleExt is not adaptive method!");
    }
 
}
```

## 4.总结
Dubbo SPI的自适应扩展(Adaptive)实现，实现了调用Adaptive方法时通过入参来选择具体的扩展实现来进行调用。理解Adaptive对阅读Dubbo源码有很大的帮助。
