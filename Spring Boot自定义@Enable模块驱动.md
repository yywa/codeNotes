#### Spring Boot自定义@Enable模块驱动

## 注解驱动

​	先看下Spring Framework已有的实现，**@EnableWebMvc**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({DelegatingWebMvcConfiguration.class})
public @interface EnableWebMvc {
}
```

​	**@Import({DelegatingWebMvcConfiguration.class})**，引用了DelegatingWebMvcConfiguration类。

```java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
  ...
}
```

​	可以看出，DelegatingWebMvcConfiguration只是一个@Configuration类。

​	接下来实现我们自己@Enable模块驱动

### 自定义@Configuration类

```java
@Configuration
public class HelloWorldConfiguration {

    @Bean
    public String helloWorld() {
        return "hello world";
    }
}
```

### 实现@EnableHelloWorld

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(HelloWorldConfiguration.class)
public @interface EnableHelloWorld {
}
```

### 自定义启动类

```java
@EnableHelloWorld
@Configuration
public class EnableHelloWorldBootStrap {
    public static void main(String[] args) {
      	//构建Spring上下文
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
      	//将当前引导类注册到Spring中
        context.register(EnableHelloWorldBootStrap.class);
      	//启动
        context.refresh();
      	//获取名称为helloWorld的Bean
        String helloWorld = context.getBean("helloWorld", String.class);
        System.out.println("helloWorld = " + helloWorld);
    }
}
```

​	这样我们就基于注解实现了模块驱动。

## 接口驱动

基于接口进行实现，需要实现ImportSelector或者ImportBeanDefinitionRegistrar接口

### 实现ImportSelector

​	假设学校里有学生和老师，Student和Teacher，通过@EnablePeople来设置人员类型。

#### 定义接口

```java
public interface People {
    void work();


    enum Type {
        /**
         * 学生
         */
        STUDENT,
        /**
         * 老师
         */
        TEACHER,
    }
}
```

#### 实现StudentPeople和TeacherPeople

```java
/**
 * 遵循约定，确保是spring的组件。
 */
@Component
public class StudentPeople implements People {
    @Override
    public void work() {
        System.out.println("这是学生在工作");
    }
}
```

```java
/**
 * 遵循约定，确保是spring的组件。
 */
@Component
public class TeacherPeople implements People {
    @Override
    public void work() {
        System.out.println("这是老师在工作");
    }
}
```

#### 定义@EnablePeople

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(PeopleImportSelector.class)
public @interface EnablePeople {
    People.Type type();
}
```

#### 实现PeopleImportSelector

```java
public class PeopleImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        //读取EnablePeople中所有的属性方法
        Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(EnablePeople.class.getName());
        People.Type type = (People.Type) annotationAttributes.get("type");
        //导入的类名称数组
        String[] importClassNames = new String[0];
        switch (type) {
            case STUDENT:
                importClassNames = new String[]{StudentPeople.class.getName()};
                break;
            case TEACHER:
                importClassNames = new String[]{TeacherPeople.class.getName()};
                break;
            default:
                break;
        }
        return importClassNames;
    }
}
```

#### 创建启动类

```java
@Configuration
@EnablePeople(type = People.Type.STUDENT)
public class BootStrap {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(BootStrap.class);
        context.refresh();
        People bean = context.getBean(People.class);
        bean.work();
    }
}
```

```
这是学生在工作
```

得到了我们想要的结果，然后将STUDENT修改为TEACHER，

```
这是老师在工作
```

### 实现ImportBeanDefinitionRegistrar

其它与实现ImportSelector一致。

#### 实现ImportBeanDefinitionRegistrar接口

```java
public class PeopleImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        //读取EnablePeople中所有的属性方法
        Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(EnablePeople.class.getName());
        People.Type type = (People.Type) annotationAttributes.get("type");
        //导入的类名称数组
        String[] importClassNames = new String[0];
        switch (type) {
            case STUDENT:
                importClassNames = new String[]{StudentPeople.class.getName()};
                break;
            case TEACHER:
                importClassNames = new String[]{TeacherPeople.class.getName()};
                break;
            default:
                break;
        }
        Stream.of(importClassNames)
                //转化为BeanDefinitionBuilder对象
                .map(BeanDefinitionBuilder::genericBeanDefinition)
                //转化为BeanDefinition
                .map(BeanDefinitionBuilder::getBeanDefinition)
                //注册到BeanDefinitionRegistry
                .forEach(beanDefinition ->
                        BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry));
    }
}
```

#### 修改@EnablePeople导入类

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(PeopleImportBeanDefinitionRegistrar.class)
public @interface EnablePeople {
    People.Type type();
}
```

重新启动，观察结果。一切正常。
