# 内部类

可以将一个类的定义放在另一个类的定义内部，这就是内部类,内部类所在的类称为外部类

内部类允许我们把一些逻辑相关的类组成在一起，并控制在内部的类的可视性，这和组合不同；并且内部类能与外部类通信。

根据定义的位置，可以将内部类分为成员内部类和局部内布类

内部类可以：

* 访问其定义所在作用域中的成员和变量，包括私有成员
* 可以对同一个包中的其他类隐藏起来
* 使用匿名内部类实现回调

## 成员内部类

成员内部类是定义在外部类成员位置的类，和成员方法，成员域等平级

所以和普通的类成员一样，成员内部类有访问权限和静态非静态之分。

### 定义和声明

创建一个内部类，只需要将内部类的定义置于外部类中:

~~~java
public class Outer {
    class Inner {
    }
    ......
}
~~~

在外部类的非静态方法中和成员变量位置，可以直接使用内部类类名`Inner`声明内部类类型:

~~~java
public class Outer {

    class Inner {

    }
    
    private Inner inner ;

    public Inner inner(){
        return new Inner();
    }
}
~~~

而在外部类的静态方法和其他类中，需要使用外部类加点加内部类`Outer.Inner`声明内部类类型，指明该内部类属于哪个外部类。

~~~java
Outer.Inner inner = new Outer().inner();
~~~

为了让其他位置能访问内部类(如果有必要的话)，外部类常常提供一个返回内部类的引用的方法，像上述例子中的`Outer.inner()`方法一样。

### 访问外部类

内部类作为外部类的成员中可以访问创建它的外部类中的其他成员:

~~~java
public class Outer {
    private int count;

    class Inner {
        public void plus(){
            count++;
        }
    }
}
~~~

非静态的内部类可以访问外部类的静态成员和非静态成员。

静态的内部类则只能访问外部类的静态成员。

另外，还可以通过外部类名紧跟着`.this`的格式在**非静态**内部类中创建外部类对象的引用:

~~~java
public class Outer {
    class Inner {
        public void test(){
            Outer outer = Outer.this;
        }
    }
}
~~~

### 创建成员内部类

**非静态内部类**

对于非静态的内部类的创建，有如下两种情况:

* 在外部类的非静态成员位置，可以直接通过`new`关键字创建内部类:

  ~~~java
  public class Outer {
      class Inner {
      }
      private Inner inner = new Inner();
      
      private void test(){
          Inner in = new Inner();
      }
  }
  ~~~

* 在外部类的静态成员位置，和其他类中，需要先创建外部类对象，在通过外部类对象变量紧跟着`.new` 的方式创建内部类对象:

  ~~~java
  Outer outer = new Outer();
  Outer.Inner inner = outer.new Inner();
  ~~~

这两种情况都表明:在拥有外部类对象前，无法创建内部类对象，这是显然的，因为内部类能访问创建它的外部类的成员。

除了直接创建内部类，其他类通常通过外部类提供的方法获取内部类(这样的内部类常常是`private`的)。这种方法常常并不会直接返回内部类的类型，而是向上转型为内部类父类的类型。

~~~java
public class Outer {
    private class Inner {
    }
    public Object getObject(){
        return new Inner();
    }
}
~~~

直接向上转型为`Object`在实践中是无意义的，这里仅为实例用。

这样对于调用该外部类的客户端，完全屏蔽了内部类的存在。

**静态内部类**

访问普通的静态成员不需要创建类对象，同样的创建静态内部类也不需要外部类

在外部类的静态成员位置，直接使用`new`关键字紧跟着静态内部类的类名创建实例:

~~~java
public class Outer {
    public static class Inner{
    }
    public static void test(){
        Inner inner = new Inner();
    }
}
~~~

而在其他类中，需要使用`new Outer.Inner()`来指示要创建的静态内部类是属于哪个外部类的:

~~~Java
Outer.Inner inner = new Outer.Inner();
~~~

## 局部内部类

成员内部类都是定义在外部类的类成员位置的，和成员方法和成员域是同级别的

而局部内部类则定义在外部类的方法和作用域内:

~~~java
public class Outer {
    public void test(){
        class PartInner{
        }
        PartInner partInner = new PartInner();
    }
}
~~~

局部内部类定义在方法和作用域内，所以它没有静态和非静态之分，这是类成员才有的区分

并且只有在作用域内才能访问到局部内部类，所以局部内部类也没有访问权限修饰符。

局部内部类可以访问外部类成员，同时也可以访问所在作用域的局部变量，不过访问的局部变量必须**事实上是`final`的**，之所以有这种限制，是因为如果访问的局部变量是可变的，那么在多线程环境下并发更新会导致竞态条件。

### 匿名内部类

我们常常使用局部内部类实现接口或者继承类，但实现或继承后的局部内常常只使用一次(作为方法的返回值、作为方法的参数传递、或者只调用了一次)

考虑如下场景，要对一个可变数组进行排序，它的排序方法接收一个`Comparator`类型的参数，为实现自定义的排序方式，我们让内部类`MyComparator`实现的`Comparator`接口,并传入了`ArrayList.sort()`方法中

~~~java
ArrayList<Integer> integers = new ArrayList<>();
integers.add(1);
integers.add(3);
integers.add(2);
class MyComparator implements Comparator<Integer> {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1 - o2;
    }
}
integers.sort(new MyComparator());
~~~

该示例中`MyComparator`被定义后只使用了一次，所以Java提供了匿名内部类的特性，将局部内部类的创建和使用放在一起:

~~~java
ArrayList<Integer> integers = new ArrayList<>();
integers.add(1);
integers.add(3);
integers.add(2);
integers.sort(new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1 - o2;
    }
});
~~~

直接在需要`Comparator`对象的位置:sort的入参位置,使用`new`和紧跟着局部内部类的父类或者父接口`Comparator`，再加上`()`紧跟着`{}`，(`{}`是匿名内部类的作用域，在其中重写方法。)

这样的语法指示直接创建一个继承自`Comparator`的匿名类的对象。

通过`new`创建的匿名内部类对象将被自动向上转型为`Comparator`类型

以上示例创建匿名内部类都是使用的默认无参构造，如果创建时需要传入参数，可以在`()`位置传入具体参数值:

~~~java
new Thread("myThread"){
    @Override
    public void run() {
        System.out.println("run");
    }
}.start();
~~~

## 闭包和回调

闭包是一个可调用的对象，它能够访问创建它的作用域的信息

可以看出内部类是面向对象的闭包，它包含外部类对象的信息，还拥有指向此外部类对象的引用。

内部类的这种特性允许它实现回调，实现调用该内部类的对象和持有该内部类的对象之间的异步通讯。

~~~java
public static void main( String[] args ) throws Exception {
    final Integer[] i = {0};
    test(new Callable<Integer>() {
        @Override
        public Integer call() throws Exception {
            return ++i[0];
        }
    });
    System.out.println(i[0]);

}

public static void test(Callable<Integer> callable) throws Exception {
    System.out.println("完成了一些事情");
    callable.call();//进行回调
}
~~~

实例中，test函数体正常是无法访问到main函数作用域中的i数组对象的，但是通过回调`Callable`，因为`Callable`的匿名内部类是在`main`中定义，所以可以访问到main方法中的对象`i`，这样test方法在合适的时机调用`Callable.call()`方法后，就间接访问到了mian方法中的数组对象`i`

## 继承成员内部类

因为成员内部类的构造器必须连接到其外围类对象的引用，所以在继承内部类的时候，如果该子类不在外部类中，则必须使用特殊的语法说明子类和其基类内部类和外部类的关系:

~~~java
public class Outer {
    class Inner{

    }
    class SubInner extends Inner{
        
    }
}
class InheritInner extends Outer.Inner{
    InheritInner(Outer outer){
        outer.super();
    }
}
~~~

内部类的子类构造器入参必须有外部类对象，且构造器必须有`outer.super()`表示子类也同样链接了其父类的外部类。

## 继承时覆盖成员内部类

在继承一个外部类时，在子类中定义同名的内部类是否能覆盖父类的内部类：

~~~java
public class Outer {
    public Outer(){
        new Inner().test();
    }

    class Inner{
        public void test(){
            System.out.println("father");
        }
    }
}
class SubOuter  extends Outer{
    class Inner{
        public void test(){
            System.out.println("sub");
        }
    }
}
class Main{
    public static void main(String[] args) {
        SubOuter subOuter = new SubOuter();
    }
}
//程序输出: father
~~~

这表明直接定义同名内部类是无法覆盖父类的内部类的，想要指定覆盖的内部类，需要：

~~~java
public class Outer {
    public Outer(){
        new Inner().test();
    }

    class Inner{
        public void test(){
            System.out.println("father");
        }
    }
}
class SubOuter  extends Outer{

    public SubOuter(){
        new Inner().test();
    }

    class Inner extends Outer.Inner{
        public void test(){
            System.out.println("sub");
        }
    }
}

class Main{
    public static void main(String[] args) {
        SubOuter subOuter = new SubOuter();
    }
}
//程序输出:
//father
//sub
~~~

# Lambda表达式

Lambda表达式是一段可被重复调用执行的代码块，可以也只能传递给函数式接口

在很多场景下能够替代匿名内部表达式作为回调的使用。

## 函数式接口

在正式了解Lambda表达式之前，有必要了解函数式接口的概念：有且只有一个抽象方法的接口被称为函数式接口

可以用注解`@FunctionalInterface`来检测注释接口是否为函数式接口

Java提供了许多函数式接口如,例如`Comparator`比较器接口:

~~~java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);
    ......
}
~~~

## Lambda表达式格式

当方法的参数中或者返回值有函数式接口类型的参数时，可以使用Lambda表达式以替代匿名内部类，或者普通的函数式接口类型实现类。

Lambda表达式的格式如下：

~~~java
(形参列表)->{重写抽象方法的方法体}
~~~

以传递给Comparator接口的Lambda表达式为例:

~~~java
Comparator<String> comparator =
(String a, String b) -> {return a.length() - b.length();};
~~~

即使 lambda 表达式没有参数， 仍然要提供空括号，就像无参数方法一样：

~~~java 
() -> { for (int i = 100;i >= 0;i-- ) System.out.println(i); }
~~~

而lambda表达式在很多情况下可以省略部分符号以简化书写:

在形参列表的参数类型已知的情况下，`()`中的数据类型可以省略：

~~~java
Comparator<String> comparator = 
    (a, b) -> {return a.length() - b.length();};
~~~

`()`中有且只有一个参数时，`()`可以省略：

~~~java
100 -> { for (int i = n;i >= 0;i++ ) System.out.println(i); } 
~~~

`{}`中只有一条语句时， `return {} ;`可以省略(必须一起省略):

~~~java
Comparator<String> comparator = 
    (a, b) ->  a.length() - b.length();
~~~

所以，如果要对一个`String`泛型的`ArrayList`容器进行排序，使用Lambda表达式进行编写将非常简便:

~~~java
//strings是类型为ArrayList<String>的对象
strings.sort((a, b) ->  a.length() - b.length());
~~~

## 方法引用

可以使用方法引用让lambda表达式传递某个指定的现成方法。

用如下格式的代码替换lambda表达式:

~~~java
object::instanceMethod; //引用对象的实例方法
Class::staticMethod;    //引用类的静态方法
Class::instanceMethod;  //引用第一个参数对象的实例方法
~~~

如：`System.out::println`就等价于`s -> System.out.println(s);`

使用示例格式中的第三种情况时，第一个参数将会成为方法的目标(方法所在的对象)例如:

~~~java
Arrays.sort(strings,(s1,s2) -> s1.compareToIgnoreCase(s2));
//可以用如下方法引用
Arrays.sort(strings,String::compareToIgnoreCase);
~~~

### `this&super`

可以在方法引用中使用`this`和`super`引用当前对象的方法或者当前对象超类的方法:

~~~java
public class App {
    public void main( ) {
        String [] strings = {"a","c","b"};
        Arrays.sort(strings,this::compareString);
    }

    public int compareString (String s1,String s2){
        return s1.length() - s2.length();
    }
}
~~~

这里的`this::compareString`就等价于`(s1,s2) -> this.compareString(s1,s2)`

### 引用构造方法

构造方法作为一种特殊的方法， 方法引用的格式和普通方法不一样，通过`new`关键字引用如:

~~~java
ArrayList<String> names = ...;
List<Person> people = names.stream().map(Person::new).collect(Collectors.toList());
~~~

这里的`Person::new`就等价于`str -> new Person(str)`
