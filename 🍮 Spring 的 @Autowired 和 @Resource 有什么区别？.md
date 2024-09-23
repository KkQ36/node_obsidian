这两个注解都是提供依赖注入功能的注解，我们来具体看一下它们的区别。
## 源码
org.springframework.beans.factory.annotation
```java
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Autowired {  
    boolean required() default true;  
}
```
其可以应用在：构造方法、方法、参数、字段、注解上。

javax.annotation
```java
@Target({ElementType.TYPE, ElementType.FIELD, ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Repeatable(Resources.class)  
public @interface Resource {  
    String name() default "";  
  
    String lookup() default "";  
  
    Class<?> type() default Object.class;  
  
    AuthenticationType authenticationType() default Resource.AuthenticationType.CONTAINER;  
  
    boolean shareable() default true;  
  
    String mappedName() default "";  
  
    String description() default "";  
  
    public static enum AuthenticationType {  
        CONTAINER,  
        APPLICATION;  
  
        private AuthenticationType() {  
        }  
    }  
}
```
其可以使用在类（接口、枚举等）、字段、方法上。
## 具体区别
### 提供者不同
通过上面的路径我们也可以看出来，`@Autowired` 是 Spring 提供的注解。
而 `@Resource` 是 javax 拓展包中的注解，其定义为 "The Resource annotation marks a resource that is needed by the application."，Spring 对这个 javax 拓展包中的注解提供了支持。
### 注入逻辑不同
`@Autowired` 只提供了按照类型去注入的能力，当然也是可以通过与其他注解，比如 `@Qualifier` 的配合实现按照名称注入。
```java
@Service  
public class TestServiceImpl {  
	// 用在字段上
	@Autowired
	UserMapper mapper;

	// 用在方法上（不只是构造方法）
	@Autowired
	public void setUserMapper(UserMapper mapper) {
		this.mapper = mapper;
	}
}
```
其默认要求注入的对象必须存在，如果可以为 NULL 的话，需要设置其 `required` 属性为 `false`（默认为 true）。
```java
@Autowired(required = false)
```
实现按照名称诸如的功能，需要使用 `@Qualifier` 注解配合：
```java
	@Autowired
	@Qualifier("userMapper1")
	UserMapper mapper;
```
`@Autowried` 在存在多个类型的 Bean 的时候，会报错，此时我们有两个方式去解决它：
1. 使用上面提到的 `@Qualifer` 注解，按照名称来进行注入。
2. 在 Bean 方法上，使用 `@Primary` 注解，来指定优先注入的 Bean。



而 `@Resource` 则原生提供了按照名称 和 按照类型 注入两种选项，它的装配顺序是这样的：
1. 如果 `name` 和 `type` 属性同时被指定，则找到唯一的 Bean 进行装配，如果找不到直接抛出异常。
2. 如果只制定了 `name`，则去寻找匹配的 Bean，找到一个或者多个都会抛出异常。
3. 如果只制定了 `type`，则去寻找匹配的 Bean，找到一个或者多个都会抛出异常。
4. 如果都没有制定，则默认使用 `byName` 的方式进行注入，也就是类名首字母小写作为 `name` 来查找 Bean。

