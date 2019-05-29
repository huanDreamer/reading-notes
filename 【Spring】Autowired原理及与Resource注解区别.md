---
title: 【Spring】Autowired原理及与Resource注解区别
date: 2019-05-17 15:03:08:008
tags: [Java, Spring] 
published: true
feature: https://user-gold-cdn.xitu.io/2018/9/9/165bc4a503dfcc18?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文转载自 [https://juejin.im/post/5cde05fae51d454759351d8c](https://juejin.im/post/5cde05fae51d454759351d8c) 

## Autowired注解

Autowired顾名思义，表示自动注入，如下是Autowired注解的源代码：
```
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {

    /**
     * Declares whether the annotated dependency is required.
     * <p>Defaults to {@code true}.
     */
    boolean required() default true;

}
```

从Autowired的实现可以看到，Autowired可以用于类的构造方法，类的字段，类的方法以及注解类型上，但是Autowired不能用于类上面。

关于Autowired注解，有如下问题需要解决：

1. Autowired作用在不同的范围上(构造方法，字段、方法)上，它的装配策略如何，按名称还是类型？

2. 为构造方法，字段和方法添加Autowired注解之后，谁来解析这个Autowired注解，完成装配

3. 装配的bean从何处而来，是在Spring的xml文件中定义的bean吗？

## 从Autowired的javadoc开始

从Autowired的javadoc中得到如下信息

1. AutowiredAnnotationBeanPostProcessor负责扫描Autowired注解，然后完成自动注入

2. 可以对私有的字段使用Autowired进行自动装配，而无需为私有字段定义getter/setter来read/write这个字段

3. 使用Autowired注解的类方法，可以是任意的方法名，任意的参数，Spring会从容器中找到合适的bean进行装配，setter自动注入跟对字段自动注入效果一样

## Autowired注解的解析

当项目中使用了Autowired注解时，需要明确的告诉Spring,配置中引用了自动注入的功能，在Spring的配置文件，做法有两种

1. 配置AutowiredAnnotationBeanPostProcessor

2. 使用[context:annotation-config/]()。[context:annotationconfig/]() 将隐式地向 Spring 容器注册`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、`PersistenceAnnotationBeanPostProcessor`以及`RequiredAnnotationBeanPostProcessor` 这 4 个 BeanPostProcessor。

## 实例

### 1. 实例一：

	* UserSerice依赖的UserDao使用Autowired注解，
	* UserDao没有在Spring配置文件中定义

**结果：UserDao为null**

### 2. 实例二：

	* UserSerice依赖的UserDao使用Autowired注解
	* UserDao在Spring配置文件中有定义

**结果：UserDao为null**

### 3. 实例三：

	* UserSerice依赖的UserDao使用Autowired注解
	* UserDao在Spring配置文件中有定义
	* Spring中使用[context:annotation-config/]()

/*/* 结果：UserDao正确注入，在Spring中配置的UserDao的实现，而在UserService中的是UserDao的接口，也就是说，虽然它们类型没有完全匹配，但是由于是实现/*/*

**关系，Spring仍然能够完成自动注入**

### 4. 实例四：

	* UserSerice依赖的UserDao使用Autowired注解
	* UserDao在Spring配置文件中有定义
	* Spring中配置AutowiredAnnotationBeanPostProcessor

**结果：UserDao正确注入，同实例三**

### 5. 实例五：

	* UserSerice依赖的UserDao使用Autowired注解
	* UserDao在Spring配置文件中有两份定义(id不同)
	* Spring中使用[context:annotation-config/]()

**结果：**

**1. 如果UserDao的属性名与某个bean的id相同，那么按照属性名和id名称匹配原则，自动装配**

**2. 如果UserService中定义的UserDao的属性名，与Spring配置文件中的两个id都不同，那么注入失败，异常抛出，提示，无法完整自动装配**

## **结论：**

1. 使用Autowired自动装配，必须在Spring的配置文件中使用[context:annotation-config/]()来告诉Spring需要进行自动装配扫描（AutowiredAnnotationBeanPostProcessor不推荐使用）

2. Autowired默认按类型进行匹配，当匹配到多个满足条件的bean时，再按照属性名和bean的id进行匹配，如果仍然有多个匹配上或者没有一个匹配上，则抛出异常，提示自动装配失败

3. 在使用Autowired时，可以使用Qualifier注解，显式的指定，当冲突发生时，使用那个id对应的bean

4. Autowired注解自动装配功能完成的是依赖的自动注入，因此，在一个bean中，它依赖的bean可以通过自动注入的方式完成而不需要显式的为它的属性进行注入。但是这些依赖的bean仍然不能省略，还是要在Spring中进行配置，省略的仅仅是bean属性的注入配置代码

## Resource注解

Resource注解在功能和目的上，等效于Autowried+Qualifier注解，Resource注解是JSR-250规范的一部分，它定义在JDK的javax.annoation包中，如下是它的定义：
```
package javax.annotation;

import java.lang.annotation.*;
import static java.lang.annotation.ElementType.*;
import static java.lang.annotation.RetentionPolicy.*;

/**
 * The Resource annotation marks a resource that is needed
 * by the application.  This annotation may be applied to an
 * application component class, or to fields or methods of the
 * component class.  When the annotation is applied to a
 * field or method, the container will inject an instance
 * of the requested resource into the application component
 * when the component is initialized.  If the annotation is
 * applied to the component class, the annotation declares a
 * resource that the application will look up at runtime. <p>
 *
 * Even though this annotation is not marked Inherited, deployment
 * tools are required to examine all superclasses of any component
 * class to discover all uses of this annotation in all superclasses.
 * All such annotation instances specify resources that are needed
 * by the application component.  Note that this annotation may
 * appear on private fields and methods of superclasses; the container
 * is required to perform injection in these cases as well.
 *
 * @since Common Annotations 1.0
 */
@Target({TYPE, FIELD, METHOD})
@Retention(RUNTIME)
public @interface Resource {
    /**
     * The JNDI name of the resource.  For field annotations,
     * the default is the field name.  For method annotations,
     * the default is the JavaBeans property name corresponding
     * to the method.  For class annotations, there is no default
     * and this must be specified.
     */
    String name() default "";

    /**
     * The name of the resource that the reference points to. It can
     * link to any compatible resource using the global JNDI names.
     *
     * @since Common Annotations 1.1
     */

    String lookup() default "";

    /**
     * The Java type of the resource.  For field annotations,
     * the default is the type of the field.  For method annotations,
     * the default is the type of the JavaBeans property.
     * For class annotations, there is no default and this must be
     * specified.
     */
    Class<?> type() default java.lang.Object.class;

    /**
     * The two possible authentication types for a resource.
     */
    enum AuthenticationType {
            CONTAINER,
            APPLICATION
    }

    /**
     * The authentication type to use for this resource.
     * This may be specified for resources representing a
     * connection factory of any supported type, and must
     * not be specified for resources of other types.
     */
    AuthenticationType authenticationType() default AuthenticationType.CONTAINER;

    /**
     * Indicates whether this resource can be shared between
     * this component and other components.
     * This may be specified for resources representing a
     * connection factory of any supported type, and must
     * not be specified for resources of other types.
     */
    boolean shareable() default true;

    /**
     * A product specific name that this resource should be mapped to.
     * The name of this resource, as defined by the <code>name</code>
     * element or defaulted, is a name that is local to the application
     * component using the resource.  (It's a name in the JNDI
     * <code>java:comp/env</code> namespace.)  Many application servers
     * provide a way to map these local names to names of resources
     * known to the application server.  This mapped name is often a
     * <i>global</i> JNDI name, but may be a name of any form. <p>
     *
     * Application servers are not required to support any particular
     * form or type of mapped name, nor the ability to use mapped names.
     * The mapped name is product-dependent and often installation-dependent.
     * No use of a mapped name is portable.
     */
    String mappedName() default "";

    /**
     * Description of this resource.  The description is expected
     * to be in the default language of the system on which the
     * application is deployed.  The description can be presented
     * to the Deployer to help in choosing the correct resource.
     */
    String description() default "";
}
```

Autowried注解，首先根据类型匹配，如果类型匹配到多个，那么在根据属性名和bean的id进行匹配(可以由Qualifier注解强制匹配指定的bean id)。Resource注解则顺序不同，它有如下几种可能的情况：

	* Resource注解指定了name属性和type属性

策略：首先进行按名称匹配策略： 匹配name属性和bean的id，如果匹配，则判断查找到的bean是否是type属性指定的类型，如果是type属性指定的类型，则匹配成功。如果不是****type属性指定的类型****，则抛出异常，提示匹配失败；如果name属性跟bean的id不匹配，则抛出异常提示没有bean的id匹配name属性

	* Resource注解指定了name属性，未指定type属性

策略：查找bean的id为name属性的bean，查找到，不关心类型为什么，都是匹配成功；如果找不到name属性指定的bean id，则匹配失败，抛出异常

	* Resource注解指定了type属性，未指定name属性

策略：首先进行按名称匹配策略： 匹配属性名和bean的id，如果匹配，则判断查找到的bean是否是type属性指定的类型，如果是type属性指定的类型，则匹配成功。如果不是****type属性指定的类型****，则抛出异常，提示匹配失败；其次进行按类型匹配策略： 如果属性名跟bean的id不匹配，则查找类型为type的bean，如果仅仅找到一个，自动装配成功，其它情况失败。

	* Resource注解未指定type属性和name属性

策略：首先进行按属性名匹配策略，匹配则注入成功；如果属性名不匹配，则进行类型匹配策略，只有为一个类型匹配才成功，其他情况都失败

### 作者注：欢迎关注笔者公号，定期分享IT互联网、金融等工作经验心得、人生感悟，欢迎交流，目前就职阿里-移动事业部，需要大厂内推的也可到公众号砸简历，或查看我个人资料获取。（公号ID：weknow619）。

![](https://user-gold-cdn.xitu.io/2018/9/9/165bc4a503dfcc18?imageView2/0/w/1280/h/960/ignore-error/1)

