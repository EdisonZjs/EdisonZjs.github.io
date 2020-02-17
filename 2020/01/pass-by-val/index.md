# 值传递？引用传递？






## 1.什么是值传递和引用传递

在探讨这个问题之前，应先搞清楚什么是值传递和引用传递以及它们之间的区别。

> 值传递：在方法被调用时，实参通过形参把它的内容副本传入方法内部，此时形参接收到的内容是实参值的一个拷贝，因此在方法内对形参的任何操作，都仅仅是对这个副本的操作，不影响原始值的内容

example :  

```java
public static void changeVal (String name,int age) {
    System.out.println("传入的name:" + name + "\n传入的age:" + age);
    name = "zjs";
    age = 24;
    System.out.println("修改后的name:" + name + "\n修改后的age:" + age);
}

public static void main (String[] args) {
    String name = "Doug lea";
    int age = 55;
    changeVal(name,age);
    System.out.println("方法调用后的name:" + name + "\n方法调用后的age:" + age);
}
```

输出：

```
传入的name:Doug lea
传入的age:55
             
修改后的name:zjs
修改后的age:24
             
方法调用后的name:Doug lea
方法调用后的age:55
```

可以看到在changesVal方法中对传入的name和age的修改，并没有影响原本的name和age的值。  

&emsp;

&emsp;

> 引用传递：”引用”也就是指向真实内容的地址值，在方法调用时，实参的地址通过方法调用被传递给相应的形参，在方法体内，形参和实参指向通愉快内存地址，对形参的操作会影响的真实内容。

example : 

```java
public static void changeVal (User user) {
    System.out.println("传入的name:" + user.getName() + "\n传入的age:" + user.getAge());
    user.setName("zjs");
    System.out.println("修改后的name:" + user.getName() + "\n修改后的age:" + user.getAge());
}

public static void main (String[] args) {
    User user = new User();
    user.setName("Doug lea");
    user.setAge(55);
    changeVal(user);
    System.out.println("方法调用后的name:" + user.getName() + "\n方法调用后的age:" + user.getAge());
}
```

输出：

```
传入的name:Doug lea
传入的age:55

修改后的name:zjs
修改后的age:55
             
方法调用后的name:zjs
方法调用后的age:55
```

可以看到changesVal方法中对传入的user的name属性修改，已经影响了原本的user的name属性的值。  

&emsp;

## 2.java中是值传递还是引用传递？

对于java是值传递还是引用传递，有不少初学者误以为java就是引用传递。他们之所以这样认为完全是被上面的第二个例子所误导。

example : 

```java
public static void changeVal (User user) {
    System.out.println("传入的name:" + user.getName() + "\n传入的age:" + user.getAge());
    user = new User();
    user.setName("zjs");
    System.out.println("修改后的name:" + user.getName() + "\n修改后的age:" + user.getAge());
}

public static void main (String[] args) {
    User user = new User();
    user.setName("Doug lea");
    user.setAge(55);
    changeVal(user);
    System.out.println("方法调用后的name:" + user.getName() + "\n方法调用后的age:" + user.getAge());
}
```

输出 :

```
传入的name:Doug lea
传入的age:55

修改后的name:zjs
修改后的age:0
             
方法调用后的name:Doug lea
方法调用后的age:55
```

可以看到，这次对user的修改，并没有影响到原本的user的值。这是为什么呢？我们只是在之前的代码中加了一行`` user = new User();``。因为把user传递给changeVal方法，传递的其实是引用的地址值的拷贝，我们之前的例子直接拿着这个地址值指向的对象做修改，当然会影响到这个对象本身。但是在加入了`` user = new User();``后，把引用变量user指向的地址值做了修改，它不再指向原本的对象，而是一个新的对象，所以对新的对象的修改，是不会影响到原本的对象。

这就说明了java中就是值传递。只不过对于引用类型，传递的这个值，是引用指向的地址值的拷贝。

可能有点绕，我们画图来解释一下。

在执行到代码``user = new User();``之前，jvm中的堆栈情况是这样的

![未命名文件1.png](https://i.loli.net/2020/01/16/ScGTkYKZqNoX4yH.png)

当执行``user = new User();``后，修改了user_copy指向的地址值

![未命名文件.png](https://i.loli.net/2020/01/16/XI7rdPZ2pzy4mg3.png)

所以从图上我们可以清除的看到java是值传递，传递给changelVal的是user指向的地址值的拷贝(0x00100)。执行``user = new User()``后，使得引用变量user重新指向一个新的地址值(new User()的地址值)。