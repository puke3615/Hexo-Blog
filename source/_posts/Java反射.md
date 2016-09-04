---
title: Java反射
date: 2016-08-29 09:46:58
tags:
    - Java
    - 反射
description: "Java是门面向对象的语言，其中的每一个对象都有与之对应的的Class。只要一提到Class，自然就想到Java的反射机制，可见反射在Java中的重要性。笔者在反射这块使用了两年多，在这里分享一些经验，不足之处，欢迎吐槽。"
---

*Java是门面向对象的语言，其中的每一个对象都有与之对应的的Class。只要一提到Class，自然就想到Java的反射机制，可见反射在Java中的重要性。笔者在反射这块使用了两年多，在这里分享一些经验，不足之处，欢迎吐槽。*

---

#####一、 Class的几种获取方式

**1. 类名.class方式**

```java
Class cls = User.class;
```

**2. 对象.getClass()方式**

```java
Class cls = user.getClass();
```

**3. Class.forName()方式**

```java
//注意这里的参数是类的全限定名(即含有包名)
Class cls = Class.forName("com.reflect.User");
```

**4. ClassLoader方式**

```java
Class cls = classLoader.findClass("com.reflect.User");
```

---

##### 二、 通过反射创建对象

**1. 通过Class创建公有的无参构造方法**

```java
Class<User> cls = User.class;
User user = cls.newInstance();
```

**2.通过Constructor创建公有的无参构造方法**

```java
Class<User> cls = User.class;
Constructor<User> constructor = cls.getConstructor();
User user = constructor.newInstance();
```

**3.通过Constructor创建私有的无参构造方法**

```java
Class<User> cls = User.class;
Constructor<User> constructor = cls.getDeclaredConstructor();
//注意，此处使用暴力反射，反射中该方法经常被使用
constructor.setAccessible(true);
User user = constructor.newInstance();
```

**4.通过Constructor创建私有的含参构造方法**

```java
Class<User> cls = User.class;
Constructor<User> constructor = cls.getDeclaredConstructor(String.class, int.class);
constructor.setAccessible(true);
//注意，此处传递的参数类型和上面的String.class和int.class保持一致
User user = constructor.newInstance("小莫", 3);
```

---

##### 三、 通过反射读写对象属性

```java
public class User {
    public static int age;
    public String name;
    private String sex;
}
```



**1. 读写静态属性**

```java
Class<User> cls = User.class;
Field ageField = cls.getField("age");
//读取静态属性
int age = (int) ageField.get(null);
//设置静态属性
ageField.set(null, 10);
```

**2. 读写成员属性**

```java
Class<User> cls = User.class;
Field nameField = cls.getField("name");
//读取成员属性
String name = (String) nameField.get(user);
//设置成员属性
nameField.set(user, "大白");
```

**3. 读写私有成员属性**

```java
Field sexField = cls.getDeclaredField("sex");
sexField.setAccessible(true);
//读取私有成员属性
String sex = (String) sexField.get(user);
//设置私有成员属性
sexField.set(user, "大白");
```

---

##### 四、 通过反射调用对象方法

```java
public class User {
    public static void say(String name) {
        System.out.println("My name is " + name);
    }
    public int plus(int a, int b) {
        return a + b;
    }
}
```

**1. 调用静态方法**

```java
Class<User> cls = User.class;
//注意，这里第一个参数是指方法名，后面的参数是可变长参数，是目标方法的对应参数类型
//若目标方法是无参的，则不填
Method sayMethod = cls.getMethod("say", String.class);
//静态方法不依赖对象实例，所以第一个参数是null
sayMethod.invoke(null, "胡巴");
```

```
后台输出: 
My name is 胡巴
```

**2. 调用成员方法**

```java
User user = new User();
Class<User> cls = User.class;
Method plusMethod = cls.getMethod("plus", int.class, int.class);
//如果该方法是非public的，则要在invoke之前调用 plusMethod.setAccessible(true);
//注意，由于该方法是成员方法，它的执行依赖于一个User实例，所以第一个参数是user
int result = (int) plusMethod.invoke(user, 2, 7);
System.out.println(result);
```

```
后台输出：
9
```

---

##### 五、 反射的简单应用

```
	上面我们对于反射中一些常用的Api进行了基本的认识，接下来我们进入一个简单的实战练习，案例驱动，以加深对反射的认识。
	场景：我们正在使用一个第三方的jar包，包含两个类User和Dog(见下)，而我们现在需求是，要调用一个User实例中dog的happy方法。
public class User {
    private Dog dog = new Dog();
}
class Dog {
    public void sleep() {
        System.out.println("the dog went to bed.");
    }
}
	
有以下难点：
	1. 这两个类都是在jar包里面的，不能直接修改
	2. 我们要有Dog类的权限，但是Dog这个类是package的
	3. 我们要拿到User实例中的dog引用，但是dog属性是private的
	4. 我们最终要调用的sleep的方法也是private的
没有反射的话，估计我们只能想到两种办法了，要么基于class字节码进行修改，要么直接找需求方互相伤害去~
废话不多说，接下来我们来看通过使用反射的方式，来KO这个需求
```

```java
//1. 获取user对象对应的Class
Class<User> userCls = User.class;

//2. 获取到改对象的dog属性
Field dogField = userCls.getDeclaredField("dog");

//3. 通过dogField属性获取user实例中对应的dog实例
//注意,由于Dog类是package级别的,所以这里无法直接声明Dog,只能使用Object代替
dogField.setAccessible(true);
Object dog = dogField.get(user);

//4. 获取dog对象对象的Class对象
Class<?> dogCls = dog.getClass();
//这里也可以使用这种方式
//Class dogCls = Class.forName("com.groovy.reflect.Dog");

//5. 获取到Dog的sleep方法
Method sleepMethod = dogCls.getDeclaredMethod("sleep");

//6. 调用sleep方法
sleepMethod.setAccessible(true);
sleepMethod.invoke(dog);
```

```
后台输出：
the dog went to bed.
```

