# 深拷贝VS浅拷贝




## 1.什么是浅拷贝

> 浅拷贝是按位拷贝，它会创建一个新的对象，这个对象有着原始对象属性值的一份精确拷贝。如果属性是基本类型，拷贝的就是基本类型的值。如果属性是内存地址，拷贝的就是内存地址，因此如果其中一个对象改变了这个地址，就会影响到另一个对象。

example ：

```java
public class Son {
    public String name;
    //...
}

public class Father implemnets Cloneable {
    private String name;
    private Son son;
    //...
    
    @Override
    protected Object clone(){
        try {
            return super.clone();
        }catch (CloneNotSupportedException e){
            e.printStackTrace();
        }
        return null;
    }
	
    public static void main (String[] args) {
    	Son son = new Son("小明");
    	Father father = new Father("大明",son);
    	Father father_copy = (Father) father.clone();
        
    	System.out.println("原始对象和浅拷贝对象是否相等: " + father == father_copy);
    	System.out.println("原始对象和浅拷贝对象的son是否相等: " + father.son = father.son);
    	
        father_copy.name = "老王";
        System.out.println("name修改的原始对象name属性: " + father.name);
        System.out.println("name修改后的拷贝对象name属性: " + father_copy.name);
        
        father_copy.son.name = "小王";
        System.out.println("son的name属性修改后,原始对象的son的name属性: " + father.son.name);
        System.out.println("son的name属性修改后,拷贝对象的son的name属性: " + father_copy.son.name);
    }
}
```

输出 :

```
原始对象和浅拷贝对象是否相等: false
原始对象和浅拷贝对象的son是否相等: true

name修改的原始对象name属性: 大明
name修改后的拷贝对象name属性: 老王

son的name属性修改后,原始对象的son的name属性: 小王
son的name属性修改后,拷贝对象的son的name属性: 小王
```

有朋友可能会疑问了，String不也是引用类型吗？为什么浅拷贝的对象name改变了，而原始对象的name却没改变？这是因为String是final修饰的常量。对于任何String类型的修改都是新建了一个String对象。所以name的修改，是new了一个新的String对象，然后被拷贝对象持有。这丝毫不会影响原始对象的name值，因为这压儿根就是两个不同的对象。

我们可以通过输出name的hashcode值来证明

```java
public static void main (String[] args) {
    Son son = new Son("小明");
    Father father = new Father("大明",son);
    Father father_copy = (Father) father.clone();
    
    System.out.println("修改前原始对象name属性的hashcode: " + father.name);
    System.out.println("修改前拷贝对象name属性的hashcode: " + father_copy.name);
 		
    father_copy.name = "老王";
    System.out.println("修改后原始对象name属性的hashcode: " + father.name);
    System.out.println("修改后拷贝对象name属性的hashcode: " + father_copy.name);
 
}
```

输出

```
修改前原始对象name属性的hashcode：756703
修改前拷贝对象name属性的hashcode：756703

修改后原始对象name属性的hashcode：756703
修改后拷贝对象name属性的hashcode：1045418
```

到这里还是有人疑问，为什么son的name属性修改了却会影响原始对象son属性的name?还记得浅拷贝的定义吗？对于原始对象的基本类型属性是拷贝值的副本，引用类型属性是拷贝一个引用。请理解清楚，``是拷贝原始对象的引用类型属性的引用，是属性的引用！``这也就是说拷贝对象和原始对象的引用并不是同一个，请参考下图。

可以看到father和copy_father完全是两个不同的对象，只是它的引用类型属性都指向同一块内存。

![1.png](https://i.loli.net/2020/01/17/t9oVLcjgm3bnJlX.png)

当我们修改copy_father的name属性，因为String是常量，所以新建一个String对象"老王"，然后copy_father的name属性的引用重新指向"老王"。

![2.png](https://i.loli.net/2020/01/17/OUfjx4MaXFtGeKD.png)

当我们修改son的name属性，新建了一个String对象“小王”，然后son的name指针重新指向"小王"

![3.png](https://i.loli.net/2020/01/17/KDAqjzsCeT9Mlih.png)

到这里，你应该能明白为什么修改copy_father的name属性不会影响原始对象father，而修改son的name属性却会影响原始对象son的name了吧。因为copy_father和father完全是两个对象，而son确是同一个啊！

下面再来看看另一段代码

```java
public static void main (String[] args) {
    Son son = new Son("小明");
    Father father = new Father("大明",son);
    Father father_copy = (Father) father.clone();
       
    father_copy.son = new Son();
    father_copy.son.name = "小王";
    System.out.println("son的name属性修改后,原始对象的son的name属性: " + father.son.name);
    System.out.println("son的name属性修改后,拷贝对象的son的name属性: " + father_copy.son.name);
}
```

输出 :

```
son的name属性修改后,原始对象的son的name属性: 小明
son的name属性修改后,拷贝对象的son的name属性: 小王
```

这是为什么？我们只不过是在修改son的name之前，加上了`` father_copy.son = new Son();``，为什么又不会影响原始son的name属性了呢？如果你有这个疑问说明你还没搞明白java中的值传递，具体了解请移步这篇文章[值传递？引用传递？](../pass-by-val)



## 2.什么是深拷贝

> 深拷贝会拷贝所有的属性，并拷贝属性指向的动态分配的内存。当对象和它所引用的对象一起拷贝时即发生深拷贝。深拷贝相比较浅拷贝速度较慢并且性能低下。

example :

```java
public class Son {
    public String name;
    //...
}

public class Father implemnets Cloneable {
    private String name;
    private Son son;
    //...
    
    @Override
    protected Object clone(){
        Son clone_son = new Son(son.name)
        Father clone_father = new Father(name,clone_son)
        return clone_father;    
    }
	
    public static void main (String[] args) {
    	Son son = new Son("小明");
    	Father father = new Father("大明",son);
    	Father father_copy = (Father) father.clone();
        
    	System.out.println("原始对象和深拷贝对象是否相等: " + father == father_copy);
    	System.out.println("原始对象和深拷贝对象的son是否相等: " + father.son = father.son);
    	
        father_copy.name = "老王";
        System.out.println("name修改的原始对象name属性: " + father.name);
        System.out.println("name修改后的拷贝对象name属性: " + father_copy.name);
        
        father_copy.son.name = "小王";
        System.out.println("son的name属性修改后,原始对象的son的name属性: " + father.son.name);
        System.out.println("son的name属性修改后,拷贝对象的son的name属性: " + father_copy.son.name);
    }
}
```

输出 :

```
原始对象和深拷贝对象是否相等: false
原始对象和深拷贝对象的son是否相等: false

name修改的原始对象name属性: 大明
name修改后的拷贝对象name属性: 老王

son的name属性修改后,原始对象的son的name属性: 小明
son的name属性修改后,拷贝对象的son的name属性: 小王
```

可以看到与浅拷贝不同的是，对于拷贝对象的引用类型属性son的name值修改，并不会影响到与原始对象。这是因为深拷贝对于引用类型不仅仅是拷贝了一个引用，而是在内存中新开辟了一块空间，把引用指向的实际内容复制了一份在新的内存空间中。

![4.png](https://i.loli.net/2020/01/17/u6GClgtSaPvyRok.png)