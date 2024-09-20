# Lombok简介

Lombok可以自动插入到编辑器和构建工具中，增强java的性能。提供简单的注解，在编译阶段自动生成JavaBean中的get，set等通用代码

优点：

* 以简单的注解形式来简化java代码，提高开发人员的开发效率

缺点：

* 高侵入性，胁迫使用
* 代码可调试性降低，可读性差
* 增加代码耦合度
* 影响版本升级
* 注解使用有风险
* 可能会破坏封装性

# 安装和依赖

* 安装IDEA Lombok插件
* 引入lombok依赖

~~~xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.14</version>
</dependency>
~~~

# 注解

## `@NonNull`

声明在方法入参参数上，被该注释声明的参数会在方法开头自动生成非空验证

~~~java
public static void print(@NonNull String message){
    System.out.println(message);
}

//          ↓↓↓↓↓↓↓↓↓↓↓编译后↓↓↓↓↓↓↓↓↓↓↓
public static void print(@NonNull String message) {
    if (message == null) {
        throw new NullPointerException("message");
    } else {
        System.out.println(message);
    }
}
~~~

## `@Cleanup`

声明在可关闭的资源对象变量上，在资源对象的上下文最后，lombok会自动调用资源对象的close方法

~~~java
public static void testClearUp()  {
    @Cleanup Scanner scanner = new Scanner(System.in);
    System.out.println(scanner.nextLine());
}
//          ↓↓↓↓↓↓↓↓↓↓↓编译后↓↓↓↓↓↓↓↓↓↓↓
public static void testClearUp() {
    Scanner scanner = new Scanner(System.in);
    try {
        System.out.println(scanner.nextLine());
    } finally {
        if (Collections.singletonList(scanner).get(0) != null) {
            scanner.close();
        }
    }
}
~~~

该注解有字段

~~~java
//清理资源调用的方法名称，默认是close，且不能有参数
String value() default "close";
~~~

## `@Getter&@Setter`

声明在类和类的字段上，lombok将自动生成get和set方法

声明在类上，如果不指定，将默认生成该类下所有字段的get和set方法

声明在属性上，将生成该字段的get和set方法

~~~java
@Getter
@Setter
public class Bean {
    private int var1;
    private String var2;
}

//          ↓↓↓↓↓↓↓↓↓↓↓编译后↓↓↓↓↓↓↓↓↓↓↓

public class Bean {
    private int var1;
    private String var2;

    public Bean() {
    }

    public int getVar1() {
        return this.var1;
    }

    public String getVar2() {
        return this.var2;
    }

    public void setVar1(int var1) {
        this.var1 = var1;
    }

    public void setVar2(String var2) {
        this.var2 = var2;
    }
}
~~~

## `@ToString`

声明在类上，lombok将自动生成该类的`toString()`方法

如果不配置：

* 默认生成的格式为：类名称后跟括号，其中包含以逗号分隔的字段

* 默认打印所有非静态字段
* 默认情况下，任何以 $ 符号开头的变量都会被自动排除
* 如果要包含的字段存在 getter，则调用它而不是使用直接字段引用

~~~java
@ToString
public class Bean {
    private int var1;
    private String var2;
}

//          ↓↓↓↓↓↓↓↓↓↓↓编译后↓↓↓↓↓↓↓↓↓↓↓

public class Bean {
    private int var1;
    private String var2;

    public Bean() {
    }

    public String toString() {
        return "Bean(var1=" + this.var1 + ", var2=" + this.var2 + ")";
    }
}
~~~

**注解源码解析**

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.SOURCE)
public @interface ToString {
    //打印时包含字段名，默认为true
    boolean includeFieldNames() default true;
    
	//在此列出的字段名，将不在toString中打印，与of互斥
    //建议使用@ToString.Exclude注解而不是该字段
    String[] exclude() default {};
    
	//将强制打印of中的字段，与exclude互斥
    //建议使用@ToString.Include注解而不是该字段
    String[] of() default {};
    
	//toString包含其父类的toString，默认为true
    boolean callSuper() default false;
    
	//默认情况下，如果要包含的字段存在 getter，toString将调用它
    //可以将该属性设置为true来关闭
    boolean doNotUseGetters() default false;
    
	//仅包含用@ToString.Include 显式标记的字段和方法
	boolean onlyExplicitlyIncluded() default false;
	
    //被该注解声明的字段将不会被@toString注解包含生成
	@Target(ElementType.FIELD)
	@Retention(RetentionPolicy.SOURCE)
	public @interface Exclude {}
	
	//被该注解声明的字段或方法将强制被@toString注解包含打印
	@Target({ElementType.FIELD, ElementType.METHOD})
	@Retention(RetentionPolicy.SOURCE)
	public @interface Include {
        
		//标记被该注解声明的字段或方法的优先级，
        //@ToString将首先打印优先级高的字段或者方法
		int rank() default 0;
        
		//自定义设置@ToString打印的字段名，默认是变量或方法名
		String name() default "";
	}
}
```

## `@EqualsAndHashCode`

在类上声明，lombok将自动生成该类的equal和hashcode方法

~~~java
@EqualsAndHashCode
public class Bean {
    private int var1;
    private String var2;
}

//          ↓↓↓↓↓↓↓↓↓↓↓编译后↓↓↓↓↓↓↓↓↓↓↓

public class Bean {
    private int var1;
    private String var2;

    public Bean() {
    }

    public boolean equals(Object o) {
        if (o == this) {
            return true;
        } else if (!(o instanceof Bean)) {
            return false;
        } else {
            Bean other = (Bean)o;
            if (!other.canEqual(this)) {
                return false;
            } else if (this.var1 != other.var1) {
                return false;
            } else {
                Object this$var2 = this.var2;
                Object other$var2 = other.var2;
                if (this$var2 == null) {
                    if (other$var2 != null) {
                        return false;
                    }
                } else if (!this$var2.equals(other$var2)) {
                    return false;
                }

                return true;
            }
        }
    }

    protected boolean canEqual(Object other) {
        return other instanceof Bean;
    }

    public int hashCode() {
        int PRIME = true;
        int result = 1;
        result = result * 59 + this.var1;
        Object $var2 = this.var2;
        result = result * 59 + ($var2 == null ? 43 : $var2.hashCode());
        return result;
    }
}

~~~

## `@NoArgsConstructor、@RequiredArgsConstructor、@AllArgsConstructor`

在类上声明该组注解，lombok将生成指定的构造器

* @NoArgsConstructor：生成一个无参构造
* @RequiredArgsConstructor：为需要特殊处理的字段生成一个构造函数，它将为下面类型的字段包含在构造器中：
    * 所有未初始化的`final`字段
    * 标记为`@NonNull`未在声明它们的地方初始化的字段

* @AllArgsConstructor 为类中的每个字段生成一个带有 1 个参数的构造函数

~~~java
@NoArgsConstructor
@RequiredArgsConstructor
@AllArgsConstructor
public class Bean {
    private int var1;
    @NonNull
    private String var2;
}

//          ↓↓↓↓↓↓↓↓↓↓↓编译后↓↓↓↓↓↓↓↓↓↓↓

public class Bean {
    private int var1;
    private @NonNull String var2;

    public Bean() {
    }

    public Bean(@NonNull String var1x) {
        if (var1x == null) {
            throw new NullPointerException("2 is marked non-null but is null");
        } else {
            this.var2 = var1x;
        }
    }

    public Bean(int var1x, @NonNull String var2x) {
        if (var2x == null) {
            throw new NullPointerException("2 is marked non-null but is null");
        } else {
            this.var1 = var1x;
            this.var2 = var2x;
        }
    }
}
~~~

## `@Date`

相当于一下所有注解的效果：

@Getter, @Setter, @ToString, @EqualsAndHashCode，@RequiredArgsConstructor

~~~java
@Data
public class Bean {
    private int var1;
    private String var2;
}
//          ↓↓↓↓↓↓↓↓↓↓↓编译后↓↓↓↓↓↓↓↓↓↓↓

public class Bean {
    private int var1;
    private String var2;

    public Bean() {
    }

    public int getVar1() {
        return this.var1;
    }

    public String getVar2() {
        return this.var2;
    }

    public void setVar1(int var1) {
        this.var1 = var1;
    }

    public void setVar2(String var2) {
        this.var2 = var2;
    }

    public boolean equals(Object o) {
        if (o == this) {
            return true;
        } else if (!(o instanceof Bean)) {
            return false;
        } else {
            Bean other = (Bean)o;
            if (!other.canEqual(this)) {
                return false;
            } else if (this.getVar1() != other.getVar1()) {
                return false;
            } else {
                Object this$var2 = this.getVar2();
                Object other$var2 = other.getVar2();
                if (this$var2 == null) {
                    if (other$var2 != null) {
                        return false;
                    }
                } else if (!this$var2.equals(other$var2)) {
                    return false;
                }

                return true;
            }
        }
    }

    protected boolean canEqual(Object other) {
        return other instanceof Bean;
    }

    public int hashCode() {
        int PRIME = true;
        int result = 1;
        result = result * 59 + this.getVar1();
        Object $var2 = this.getVar2();
        result = result * 59 + ($var2 == null ? 43 : $var2.hashCode());
        return result;
    }

    public String toString() {
        return "Bean(var1=" + this.getVar1() + ", var2=" + this.getVar2() + ")";
    }
}
~~~





