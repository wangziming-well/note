# PropertyEditor

Spring使用 java内置的`PropertyEditor` 来实现 `Object` 和 `String` 之间的转换。它属于JavaBean规范。`PropertyEditor`属性编辑器的主要功能就是将外部的设置值转换为JVM内部的对应类型，所以属性编辑器其实就是一个类型转换器。它主要通过下面方法实现外部设置值(String字符串)和类Class的转换：

~~~java
public interface PropertyEditor {
	void setValue(Object value); //以Class形式设置属性
    Object getValue(); //以Class形式获取属性
    String getAsText(); //以String形式获取属性
    void setAsText(String text) throws java.lang.IllegalArgumentException; //以String形式设置属性
}
~~~

例如想要通过`ClassEditor`实现`String`到`Class`的转换，可以先调用`setAsText()`再调用`getValue()`：

~~~java
ClassEditor classEditor = new ClassEditor();
classEditor.setAsText("org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver");
Class value =(Class) classEditor.getValue();
System.out.println(value);
~~~

类似的，要实现`Class`到`String`的转换，就先调用`setValue()`再调用`getAsText()`。

在Spring中可能在以下场景使用`PropertyEditor`：

* 通过`PropertyEditor`在Bean上设置属性，我们在Spring的XML配置文件上使用`String`字符串为bean对象设置属性，那么Spring会尝试用`PropertyEditor`将字符串转换为需要的属性
* 在Spring MVC中，解析HTTP请求参数(字符串)是通过使用各种 `PropertyEditor` 实现完成

Spring在`org.springframework.beans.propertyeditors`包中内置了许多 `PropertyEditor` 实现：

| 类                        | 说明                                                         |
| :------------------------ | :----------------------------------------------------------- |
| `ByteArrayPropertyEditor` | 字节数组的编辑器。将字符串转换为其相应的字节表示。默认由 `BeanWrapperImpl` 注册。 |
| `ClassEditor`             | 将代表类的字符串解析为实际的类，反之亦然。当没有找到一个类时，会抛出一个 `IllegalArgumentException`。默认情况下，通过 `BeanWrapperImpl` 注册。 |
| `CustomBooleanEditor`     | 用于 `Boolean` 属性的可定制的属性编辑器。默认情况下，由 `BeanWrapperImpl` 注册，但可以通过注册它的一个自定义实例作为自定义编辑器来重写。 |
| `CustomCollectionEditor`  | 集合的属性编辑器，将任何源 `Collection` 转换为一个给定的目标 `Collection` 类型。 |
| `CustomDateEditor`        | `java.util.Date` 的可定制属性编辑器，支持自定义 `DateFormat`。默认情况下没有注册。必须根据需要由用户注册适当的格式（format）。 |
| `CustomNumberEditor`      | 可定制的属性编辑器，用于任何 `Number` 子类，如 `Integer`、`Long`、`Float` 或 `Double`。默认情况下，由 `BeanWrapperImpl` 注册，但可以通过注册它的一个自定义实例作为自定义编辑器来重写。 |
| `FileEditor`              | 将字符串解析为 `java.io.File` 对象。默认情况下，由 `BeanWrapperImpl` 注册。 |
| `InputStreamEditor`       | 单向属性编辑器，可以接受一个字符串并产生（通过中间的 `ResourceEditor` 和 `Resource`）一个 `InputStream`，这样 `InputStream` 属性可以直接设置为字符串。注意，默认用法不会为你关闭 `InputStream`。默认情况下，由 `BeanWrapperImpl` 注册。 |
| `LocaleEditor`            | 可以将字符串解析为 `Locale` 对象，反之亦然（字符串格式为 `[language]_[country]_[variant]`，与 `Locale` 的 `toString()` 方法相同）。也接受空格作为分隔符，作为下划线的替代。默认情况下，由 `BeanWrapperImpl` 注册。 |
| `PatternEditor`           | 可以将字符串解析为 `java.util.regex.Pattern` 对象，反之亦然。 |
| `PropertiesEditor`        | 可以将字符串（格式为 `java.util.Properties` 类的javadoc中定义的格式）转换为 `Properties` 对象。默认情况下，由 `BeanWrapperImpl` 注册。 |
| `StringTrimmerEditor`     | trim 字符串的属性编辑器。可选择允许将空字符串转换为 `null` 值。默认情况下没有注册 - 必须由用户注册。 |
| `URLEditor`               | 可以将一个 URL 的字符串表示解析为一个实际的 `URL` 对象。默认情况下，由 `BeanWrapperImpl` 注册。 |

如果如果 `PropertyEditor` 类与它们所处理的类在同一个包中，并且 `PropertyEditor`名称是该类名后缀一个 `Editor`，那么Spring会自动发现这些类。例如：如果有以下的类和包结构，那么可以让 `SomethingEditor` 类被识别并作为 `Something` 类型属性的 `PropertyEditor` 使用：

~~~java
com
  chank
    pop
      Something
      SomethingEditor // the PropertyEditor for the Something class
~~~

## 注册自定义`PropertyEditor`

在将配置字符串设置为Bean属性时， Spring IoC容器会使用标准JavaBeans `PropertyEditor` 实现把这些字符串转换成属性的复杂类型。Spring 预先注册了一些自定义的`PropertyEditor`（例如，将表示为字符串的类名转换为 `Class` 对象）

若要注册其他自定义的 `PropertyEditors`，一种方法时手动调用`ConfigurableBeanFactory.registerCustomEditor()`注册，这种方法通常不方便，也不推荐。

### 使用`CustomEditorConfigurer`

稍微方便一些的方式是使用`CustomEditorConfigurer`，它是一个`BeanFactoryPostProcessor`，可以在容器初始化时注册指定的`PropertyEditors`

一个注册`PropertyEditor`自定义的例子,考虑下面Bean类：

~~~java
package example;

public class ExoticType {

    private String name;

    public ExoticType(String name) {
        this.name = name;
    }
}

public class DependsOnExoticType {

    private ExoticType type;

    public void setType(ExoticType type) {
        this.type = type;
    }
}
~~~

如果需要注册一个`DependsOnExoticType`,需要下面配置：

~~~xml
<bean id="sample" class="example.DependsOnExoticType">
    <property name="type" >
        <bean class="example.ExoticType">
            <property name="name" value="aNameForExoticType" />
        </bean>
    </property>
</bean>
~~~

为了简化这种配置，可以实现一个`ExoticType`对应的`PropertyEditor`:

~~~java
package example;

import java.beans.PropertyEditorSupport;

// converts string representation to ExoticType object
public class ExoticTypeEditor extends PropertyEditorSupport {

    public void setAsText(String text) {
        setValue(new ExoticType(text.toUpperCase()));
    }
}
~~~

将其通过`CustomEditorConfigurer`注册到容器中：

~~~xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="customEditors">
        <map>
            <entry key="example.ExoticType" value="example.ExoticTypeEditor"/>
        </map>
    </property>
</bean>
~~~

这样此时想要注册一个`DependsOnExoticType`,只需要下面配置：

~~~xml
<bean id="sample" class="example.DependsOnExoticType">
    <property name="type" value="aNameForExoticType"/>
</bean>
~~~

### 使用 `PropertyEditorRegistrar`

另一种在Spring 容器中注册`PropertyEditor`的机制是创建和使用`PropertyEditorRegistrar`,特别是需要在几种不同的情况下使用同一组`PropertyEditor`的场景下，这种机制会很有用。此时可以实现一个`PropertyEditorRegistrar`，并在多种情况下重复使用这个注册器。

这个`PropertyEditorRegistrar`注册器和 `PropertyEditorRegistry`注册表协同工作。`PropertyEditorRegistrar`负责向`PropertyEditorRegistry`中注册指定的多个`PropertyEditor`。`BeanWrapper`和`DataBinder`都实现了`PropertyEditorRegistry`,都是属性编辑器的注册表。

可以通过`CustomEditorConfigurer.setPropertyEditorRegistrars(..)`向其中设置一个`PropertyEditorRegistrar` 实例。以这种方式添加到 `CustomEditorConfigurer` 的 `PropertyEditorRegistrar` 实例可以很容易地与 `DataBinder` 和Spring MVC控制器共享。

此外若使用 `PropertyEditorRegistrar` ，那么`PropertyEditor`就不需要考虑同步情况，因为 `PropertyEditorRegistrar` 会为每个bean build 创建新的 `PropertyEditor` 实例。

一个自定义 `PropertyEditorRegistrar` 的实现示例：

~~~java
package com.foo.editors.spring;

public final class CustomPropertyEditorRegistrar implements PropertyEditorRegistrar {

    public void registerCustomEditors(PropertyEditorRegistry registry) {

        // it is expected that new PropertyEditor instances are created
        registry.registerCustomEditor(ExoticType.class, new ExoticTypeEditor());

        // you could register as many custom property editors as are required here...
    }
}
~~~

下面例子通过 `CustomEditorConfigurer`，来配置 `CustomPropertyEditorRegistrar` 实例，以配置到容器：

~~~xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="propertyEditorRegistrars">
        <list>
            <ref bean="customPropertyEditorRegistrar"/>
        </list>
    </property>
</bean>
<bean id="customPropertyEditorRegistrar" class="com.foo.editors.spring.CustomPropertyEditorRegistrar"/>
~~~

# Spring类型转换

`org.springframework.core.convert`包提供了一个通用的类型转换框架。在Spring容器中，这个框架可以作为 `PropertyEditor` 实现的替代，将外部配置的字符串转换为Bean所需的属性类型。

## `Converter`

Spring类型转换的核心接口是`Converter`，其定义如下：

~~~java
package org.springframework.core.convert.converter;

public interface Converter<S, T> {

    T convert(S source);
}
~~~

其中泛型`S`为转换的原类型，泛型`T`为转换的目标类型。

 `StringToInteger` 类，它是一个典型的 `Converter` 实现：

~~~java
package org.springframework.core.convert.support;

final class StringToInteger implements Converter<String, Integer> {

    public Integer convert(String source) {
        return Integer.valueOf(source);
    }
}
~~~

## `ConverterFactory`

如果需要整个类层次结构的转换逻辑时（例如，当从 `String` 转换到 `Enum` 对象时），可以实现`ConverterFactory`，其定义为：

~~~java
package org.springframework.core.convert.converter;

public interface ConverterFactory<S, R> {

    <T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
~~~

泛型`S`要转换的类型，`R` 是定义可以转换的类的范围的基类型。实现 `getConverter(Class<T>)`，其中T是R的一个子类。

以 `StringToEnumConverterFactory` 为例：

~~~java
package org.springframework.core.convert.support;

final class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

    public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
        return new StringToEnumConverter(targetType);
    }

    private final class StringToEnumConverter<T extends Enum> implements Converter<String, T> {

        private Class<T> enumType;

        public StringToEnumConverter(Class<T> enumType) {
            this.enumType = enumType;
        }

        public T convert(String source) {
            return (T) Enum.valueOf(this.enumType, source.trim());
        }
    }
}
~~~

## `GenericConverter`

 `GenericConverter` 接口提供更复杂，更通用的 `Converter` 实现。 `GenericConverter` 相比 `Converter` 更灵活并且没有泛型。其定义如下：

~~~java
package org.springframework.core.convert.converter;

public interface GenericConverter {

    public Set<ConvertiblePair> getConvertibleTypes();

    Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
~~~

 `getConvertibleTypes()` 返回当前转换器支持的源类型 → 目标类型对。

`convert(Object, TypeDescriptor, TypeDescriptor)`完成具体的转换逻辑，`TypeDescriptor`是对field字段的类型Class信息的描述符。 sourceType表示source的类型，targetType指示要转换成的类型。

## ``ConditionalGenericConverter``

如果需要让一个 `Converter` 只在一个特定的条件成立的情况下运行。例如，可能想只在目标字段上存在特定注解的情况下运行一个 `Converter`，或者可能只需要在目标类上定义了特定的方法（比如 `static valueOf` 方法）的情况下运行一个 `Converter`

这种情况下可以使用 `ConditionalGenericConverter` ,其定义如下：

~~~java
public interface ConditionalConverter {

    boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
}

public interface ConditionalGenericConverter extends GenericConverter, ConditionalConverter {
}
~~~

## `ConversionService`

`ConversionService`提供一个统一的类型转换服务。它会将类型转换的请求委托给内部的`Converter `执行，其定义如下：

~~~java
package org.springframework.core.convert;

public interface ConversionService {

    boolean canConvert(Class<?> sourceType, Class<?> targetType);

    <T> T convert(Object source, Class<T> targetType);

    boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);

    Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
~~~

大多数 `ConversionService` 实现也实现了 `ConverterRegistry`，真正工作前它会被注册多个`Converter`。`ConversionService` 的实现会委托其注册的`Converter`来执行类型转换逻辑

`GenericConversionService` 是一个通用的实现

 `ConversionServiceFactory` 提供了一个方便的工厂来创建常见的 `ConversionService` 配置

## 配置`ConversionService`

`ConversionService` 是一个无状态的对象，应用在启动时会实例化它，多个线程会共享这个实例。

在Spring应用中通常需要为每个Spring容器配置一个 `ConversionService` 实例。在需要的时候，Spring就会使用它进行类型转换。

如果没有向Spring注册 `ConversionService`，就会使用基于 `PropertyEditor` 的类型抓获。

要向Spring注册一个默认的 `ConversionService`，就必须使用 `id` 为 `conversionService`的bean，例如：

~~~xml
<bean id="conversionService"
    class="org.springframework.context.support.ConversionServiceFactoryBean"/>
~~~

`ConversionServiceFactoryBean`会生产一个`DefaultConversionService`,这个默认`ConversionService`可以在字符串、数字、枚举、集合、map和其他常见类型之间进行转换。如果需要添加自定义的`Converter`或者覆盖默认的`Converter`，可以设置`converters`属性，属性类型可以是 `Converter`、`ConverterFactory` 或 `GenericConverter` ，例如：

~~~xml
<bean id="conversionService"
        class="org.springframework.context.support.ConversionServiceFactoryBean">
    <property name="converters">
        <set>
            <bean class="example.MyCustomConverter"/>
        </set>
    </property>
</bean>
~~~

## 使用`ConversionService`

如果需要使用`ConversionService`提供的类型转换服务，可以直接注入到需要的bean中，例如：

~~~java
@Service
public class MyService {

    public MyService(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    public void doIt() {
        this.conversionService.convert(...)
    }
}
~~~

大多数情况下，只需要使用指定`targetType`的`convert()`方法就能完成转换：

~~~java
String source = "2023-08-27";
LocalDate convert =(LocalDate) conversionService.convert(source, TypeDescriptor.valueOf(LocalDate.class));
~~~

但是对于更复杂的类型，如一个泛型的集合，例如将 `List<Integer>` 转换成 `List<String>`，就需要提供源和目标类型的正式定义，`TypeDescriptor`提供了相关的静态方法：

~~~java
List<String> input = Arrays.asList("2024-02-21","2024-02-22","2024-02-23");
List<LocalDate> convert =(List<LocalDate>) conversionService.convert(input,
        TypeDescriptor.forObject(input), // List<Integer> type descriptor
        TypeDescriptor.collection(List.class, TypeDescriptor.valueOf(LocalDate.class)));
System.out.println(convert);
~~~

# Spring字段格式化

在一些场景中，例如解析HTTP请求，读取外部配置，常常需要将`String`字符串转换为其他类型。而且经常需要对`String`值进行本地化。Spring提供了 `Formatter` 接口以解决这样的需求；会使用 `Formatter` 来实现`Converter`。定义如下：

~~~java
package org.springframework.format;

public interface Formatter<T> extends Printer<T>, Parser<T> {
}
public interface Printer<T> {

    String print(T fieldValue, Locale locale);
}
public interface Parser<T> {

    T parse(String clientValue, Locale locale) throws ParseException;
}
~~~

泛型`T`是希望格式化的对象类型。`print()`将对象转化为字符串，`parse()`将字符串解析为对象。

`format` 子包提供了几个 `Formatter` 的实现：

* 格式化`Number`
  *  `NumberStyleFormatter`
  * `CurrencyStyleFormatter `
  *  `PercentStyleFormatter`
* 格式化`Date`
  *  `DateFormatter`

## 注解驱动的格式化

可以通过字段类型或注解来配置字段格式化，`AnnotationFormatterFactory`可以将注解绑定到`Formatter`,定义如下：

~~~java
package org.springframework.format;

public interface AnnotationFormatterFactory<A extends Annotation> {

    Set<Class<?>> getFieldTypes();

    Printer<?> getPrinter(A annotation, Class<?> fieldType);

    Parser<?> getParser(A annotation, Class<?> fieldType);
}
~~~

其中泛型`A`为希望与格式化关联的注解类型，例如`org.springframework.format.annotation.DateTimeFormat`

 `getFieldTypes()` 返回注解可以使用的字段类型。 `getPrinter()` 返回一个 `Printer` 来将注解字段转化为字符串。 `getParser()` 返回一个 `Parser` 来将字符串转换为注解字段类型。

下面示例一个 `AnnotationFormatterFactory` 的实现，将 `@NumberFormat` 注解绑定到一个 formatter 上：

~~~java
public final class NumberFormatAnnotationFormatterFactory
        implements AnnotationFormatterFactory<NumberFormat> {

    private static final Set<Class<?>> FIELD_TYPES = Set.of(Short.class,
            Integer.class, Long.class, Float.class, Double.class,
            BigDecimal.class, BigInteger.class);

    public Set<Class<?>> getFieldTypes() {
        return FIELD_TYPES;
    }

    public Printer<Number> getPrinter(NumberFormat annotation, Class<?> fieldType) {
        return configureFormatterFrom(annotation, fieldType);
    }

    public Parser<Number> getParser(NumberFormat annotation, Class<?> fieldType) {
        return configureFormatterFrom(annotation, fieldType);
    }

    private Formatter<Number> configureFormatterFrom(NumberFormat annotation, Class<?> fieldType) {
        if (!annotation.pattern().isEmpty()) {
            return new NumberStyleFormatter(annotation.pattern());
        }
        // else
        return switch(annotation.style()) {
            case Style.PERCENT -> new PercentStyleFormatter();
            case Style.CURRENCY -> new CurrencyStyleFormatter();
            default -> new NumberStyleFormatter();
        };
    }
}
~~~

此时可以用 `@NumberFormat` 来注解字段以触发格式化：

~~~java
public class MyModel {

    @NumberFormat(style=Style.CURRENCY)
    private BigDecimal decimal;
}
~~~

### 可用格式化注解

`org.springframework.format.annotation `包提供两个格式化注解：

*  `@DateTimeFormat` 用来格式化`Date`、`Calendar`、`Long`(时间戳)和`java.time`包下类型
*  `@NumberFormat` 用来格式化`Number`字段

一个使用 `@DateTimeFormat` 将一个 `java.util.Date` 格式化为ISO日期（yyyy-MM-dd）的示例：

~~~java
public class MyModel {

    @DateTimeFormat(iso=ISO.DATE)
    private Date date;
}
~~~

## `FormatterRegistry`

`FormatterRegistry` 是一个用于注册formatter和converter的SPI

 `FormattingConversionService` 是 `FormatterRegistry` 的一个实现，适用于大多数环境。它同时也实现了 `ConversionService`。

使用 `FormattingConversionServiceFactoryBean`，可以通过编程或声明的方式将 `FormattingConversionService`配置为Spring Bean

下面是 `FormattingConversionService` 的定义：

~~~java
package org.springframework.format;

public interface FormatterRegistry extends ConverterRegistry {

    void addPrinter(Printer<?> printer);

    void addParser(Parser<?> parser);

    void addFormatter(Formatter<?> formatter);

    void addFormatterForFieldType(Class<?> fieldType, Formatter<?> formatter);

    void addFormatterForFieldType(Class<?> fieldType, Printer<?> printer, Parser<?> parser);

    void addFormatterForFieldAnnotation(AnnotationFormatterFactory<? extends Annotation> annotationFormatterFactory);
}

~~~

有了`FormatterRegistry`就可以集中配置Formatter

## `FormatterRegistrar` 

`FormatterRegistrar `提供向`FormatterRegistry`注册formatter和converter的接口，下面是定义：

~~~java
package org.springframework.format;

public interface FormatterRegistrar {

    void registerFormatters(FormatterRegistry registry);
}
~~~

通过 `FormatterRegistrar` ，可以方便的实现为一个format类型注册多个conveter和formatter。

## 配置全局Date和Time格式

默认情况下，没有被 `@DateTimeFormat` 注解注释的date和time字段会通过使用 `DateFormat.SHORT` 样式从字符串转换。

我们可以定义全局格式来改变这一默认行为，为此需要使用下面注册器手动注册formatter以覆盖默认的formatter：

- `org.springframework.format.datetime.standard.DateTimeFormatterRegistrar`
- `org.springframework.format.datetime.DateFormatterRegistrar`

例如，下面的 Java 配置注册了一个全局 `YYYMMDD` 格式：

~~~java
@Configuration
public class AppConfig {

    @Bean
    public FormattingConversionService conversionService() {

        // Use the DefaultFormattingConversionService but do not register defaults
        DefaultFormattingConversionService conversionService =
            new DefaultFormattingConversionService(false);

        // Ensure @NumberFormat is still supported
        conversionService.addFormatterForFieldAnnotation(
            new NumberFormatAnnotationFormatterFactory());

        // Register JSR-310 date conversion with a specific global format
        DateTimeFormatterRegistrar dateTimeRegistrar = new DateTimeFormatterRegistrar();
        dateTimeRegistrar.setDateFormatter(DateTimeFormatter.ofPattern("yyyyMMdd"));
        dateTimeRegistrar.registerFormatters(conversionService);

        // Register date conversion with a specific global format
        DateFormatterRegistrar dateRegistrar = new DateFormatterRegistrar();
        dateRegistrar.setFormatter(new DateFormatter("yyyyMMdd"));
        dateRegistrar.registerFormatters(conversionService);

        return conversionService;
    }
}
~~~

或者使用一个 `FormattingConversionServiceFactoryBean`完成基于XML的配置：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="conversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
        <property name="registerDefaultFormatters" value="false" />
        <property name="formatters">
            <set>
                <bean class="org.springframework.format.number.NumberFormatAnnotationFormatterFactory" />
            </set>
        </property>
        <property name="formatterRegistrars">
            <set>
                <bean class="org.springframework.format.datetime.standard.DateTimeFormatterRegistrar">
                    <property name="dateFormatter">
                        <bean class="org.springframework.format.datetime.standard.DateTimeFormatterFactoryBean">
                            <property name="pattern" value="yyyyMMdd"/>
                        </bean>
                    </property>
                </bean>
            </set>
        </property>
    </bean>
</beans>
~~~

# Spring 验证

Spring提供一个 `org.springframework.validation.Validator` 接口，它通过下面两个方法为类提供验证：

- `supports(Class)`: 指示提供的`Class`实例是否可以被这个 `Validator` 验证
- `validate(Object, org.springframework.validation.Errors)`: 验证给定的对象，如果出现验证不通过，则用给定的 `Errors` 对象登记不通过原因。

考虑下面对象：

~~~java
public class Person {
    private String name;
    private int age;
    // the usual getters and setters...
}
~~~

实现一个对`Person`实例验证的 `Validator`：

~~~java
public class PersonValidator implements Validator {

    /**
     * This Validator validates only Person instances
     */
    public boolean supports(Class clazz) {
        return Person.class.equals(clazz);
    }

    public void validate(Object obj, Errors e) {
        ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
        Person p = (Person) obj;
        if (p.getAge() < 0) {
            e.rejectValue("age", "negativevalue");
        } else if (p.getAge() > 110) {
            e.rejectValue("age", "too.darn.old");
        }
    }
}
~~~

我们可以通过`Errors`的多个重载方法`reject()`和`rejectValue()`方法登记拒绝信息；`rejectValue()`表示对象的某个指定字段验证不通过，而`reject()`表示整个对象全局验证不通过。

`ValidationUtils`工具类为我们验证时登记拒绝信息提供方便，例如 `rejectIfEmpty(..)` 在指定属性为`null`或者空字符串时进行拒绝。

对于有对象属性的对象，即嵌套对象。当然可以用一个`Validator`来验证它，但是最好为每个嵌套类对象单独设计一个`Validator`实现。

例如对于下面嵌套对象：

~~~java
public class Customer {
    String firstName;
    String surname;
    Address address;
    //......
}
~~~

其中`Address`也是一个独立对象，并且已经有一个独立的 `AddressValidator`实现。可以在 `CustomerValidator` 中重用 `AddressValidator` ，具体方法如下：

~~~java
public class CustomerValidator implements Validator {

    private final Validator addressValidator;

    public CustomerValidator(Validator addressValidator) {
        if (addressValidator == null) {
            throw new IllegalArgumentException("The supplied [Validator] is " +
                "required and must not be null.");
        }
        if (!addressValidator.supports(Address.class)) {
            throw new IllegalArgumentException("The supplied [Validator] must " +
                "support the validation of [Address] instances.");
        }
        this.addressValidator = addressValidator;
    }

    /**
     * This Validator validates Customer instances, and any subclasses of Customer too
     */
    public boolean supports(Class clazz) {
        return Customer.class.isAssignableFrom(clazz);
    }

    public void validate(Object target, Errors errors) {
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "field.required");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "surname", "field.required");
        Customer customer = (Customer) target;
        try {
            errors.pushNestedPath("address"); //将路径上下文切换到子路径address上
            ValidationUtils.invokeValidator(this.addressValidator, customer.getAddress(), errors); //使用addressValidator对属性address验证
        } finally {
            errors.popNestedPath(); //将路径上下文切回到父路径
        }
    }
}
~~~

# JSR-303数据校验

JSR-303是Java为Bean数据合法性校验提供的标准框架。`hibernate-validator`包是对该框架的实现，可以引入它来进行数据校验:

~~~xml
<!-- 实现了jdk的Validator、ConstraintValidator、Constraint接口，提供校验实现 -->
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.1.5.Final</version>
</dependency>
~~~

Bean Validation为Java应用程序提供了一种通过约束声明和元数据进行验证的通用方法。可以用声明性的验证约束（constraint）来注解domain模型属性，然后由运行时强制执行。Java提供有内置的约束，也允许自定义约束。例如：

~~~java
public class PersonForm {

    @NotNull
    @Size(max=64)
    private String name;

    @Min(0)
    private int age;
}
~~~

Bean Validation 验证器根据声明的约束条件来验证该类的实例。



JSR-303定义了如下注解规范:

| 注解                        | 代码内容                                                 |
| --------------------------- | -------------------------------------------------------- |
| @Null                       | 被注释的元素必须为 null                                  |
| @NotNull                    | 被注释的元素必须不为 null                                |
| @AssertTrue                 | 被注释的元素必须为 true                                  |
| @AssertFalse                | 被注释的元素必须为 false                                 |
| @Min(value)                 | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @Max(value)                 | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @DecimalMin(value)          | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @DecimalMax(value)          | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @Size(max, min)             | 被注释的元素的大小必须在指定的范围内                     |
| @Digits (integer, fraction) | 被注释的元素必须是一个数字，其值必须在可接受的范围内     |
| @Past                       | 被注释的元素必须是一个过去的日期                         |
| @Future                     | 被注释的元素必须是一个将来的日期                         |
| @Pattern(value)             | 被注释的元素必须符合指定的正则表达式                     |

hibernate-validator提供的额外注解:

| @代码     | 代码内容                               |
| --------- | -------------------------------------- |
| @Email    | 被注释的元素必须是电子邮箱地址         |
| @Length   | 被注释的字符串的大小必须在指定的范围内 |
| @NotEmpty | 被注释的字符串的必须非空               |
| @Range    | 被注释的元素必须在合适的范围内         |

# Spring支持JSR-303

## 配置Bean Validation Provider

Spring提供了对Bean Validation API的全面支持，以在应用中需要验证的地方注入一个 `jakarta.validation.ValidatorFactory` 或 `jakarta.validation.Validator`。

以使用 `LocalValidatorFactoryBean` 来配置一个默认的 `Validator` 作为Spring Bean，如下例所示：

~~~java
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;

@Configuration
public class AppConfig {

    @Bean
    public LocalValidatorFactoryBean validator() {
        return new LocalValidatorFactoryBean();
    }
}
~~~

上边配置会通过使用默认的引导机制触发Bean `Validation` 的初始化。会自动检测到存在于classpath中的Bean `Validation` provider

## 使用Validator

`LocalValidatorFactoryBean` 同时实现了 `jakarta.validation.ValidatorFactory` 和 `jakarta.validation.Validator`，以及Spring的 `org.springframework.validation.Validator`。

所以在调用验证逻辑时可以直接将其注入到对应的Bean中。

可以以注入对 `jakarta.validation.Validator` 的引用：

~~~java
import jakarta.validation.Validator;

@Service
public class MyService {

    @Autowired
    private Validator validator;
}
~~~

也可以注入对 `org.springframework.validation.Validator` 的引用：

~~~java
import org.springframework.validation.Validator;

@Service
public class MyService {

    @Autowired
    private Validator validator;
}
~~~

## 配置自定义约束

每个bean验证约束由两部分组成。

- 一个 `@Constraint` 注解，声明了约束及其可配置的属性。
- `jakarta.validation.ConstraintValidator` 接口的一个实现，实现约束的行为。

将声明与实现联系起来，每个 `@Constraint` 注解都会引用一个相应的 `ConstraintValidator` 实现类。在运行时，当你的domain模型中遇到约束注解时， `ConstraintValidatorFactory` 会实例化所引用的实现

默认情况下，`LocalValidatorFactoryBean` 配置了一个 `SpringConstraintValidatorFactory`，使用Spring来创建 `ConstraintValidator` 实例。可以让自定义 `ConstraintValidators` 像其他 Spring Bean 使用依赖注入。

下面的例子显示了一个自定义的 `@Constraint` 声明和一个相关的 `ConstraintValidator` 实现，并使用依赖注入：

~~~java
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy=MyConstraintValidator.class)
public @interface MyConstraint {
}
~~~

`ConstraintValidator`实现:

~~~java
import jakarta.validation.ConstraintValidator;

public class MyConstraintValidator implements ConstraintValidator {

    @Autowired;
    private Foo aDependency;

    // ...
}
~~~

可以看到在`ConstraintValidator`中可以实现依赖注入使用 `@Autowired`

## Spring驱动的方法验证

可以通过 `MethodValidationPostProcessor` Bean定义将Bean Validation 1.1支持的方法验证功能集成到Spring context中

~~~java
import org.springframework.validation.beanvalidation.MethodValidationPostProcessor;

@Configuration
public class AppConfig {

    @Bean
    public MethodValidationPostProcessor validationPostProcessor() {
        return new MethodValidationPostProcessor();
    }
}
~~~

如果需要方法验证，可以在目标 类使用Spring的 `@Validated` 注解

# Spring数据绑定

Spring数据绑定体系是以`DataBinder`为基础设施开展的。`DataBinder`提供数据绑定和数据校验服务。

使用下面domain类进行数据绑定的演示：

~~~java
import lombok.Data;
import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;

@Data
public class People {
    @NotNull
    public String name;
    @Min(0)
    public String age;
    public People father;
    public People mother;

}
~~~

使用`DataBinder`进行数据绑定：

~~~java
People people = new People();
MutablePropertyValues mutablePropertyValues = new MutablePropertyValues();
mutablePropertyValues.add("name","狗蛋儿");
mutablePropertyValues.add("age",18);
mutablePropertyValues.add("mother.name","小朋友");
mutablePropertyValues.add("mother.age",20);
mutablePropertyValues.add("father.name","大忽悠");
mutablePropertyValues.add("father.age",20);
DataBinder dataBinder = new DataBinder(people);
dataBinder.bind(mutablePropertyValues);
dataBinder.close();
System.out.println("数据绑定结果为: "+people);
~~~

使用`DataBinder`进行数据校验：

~~~java
@SpringBootApplication
public class SpringdemoApplication {

	public static void main(String[] args) {
		ConfigurableApplicationContext context = SpringApplication.run(SpringdemoApplication.class, args);
		LocalValidatorFactoryBean bean = context.getBean(LocalValidatorFactoryBean.class);
		People people = new People();
		MutablePropertyValues mutablePropertyValues = new MutablePropertyValues();
		mutablePropertyValues.add("name","狗蛋儿");
		mutablePropertyValues.add("age",-5);

		DataBinder dataBinder = new DataBinder(people);
		dataBinder.setValidator(bean);
		dataBinder.bind(mutablePropertyValues);
		dataBinder.validate();
		BindingResult bindingResult = dataBinder.getBindingResult();
		System.out.println("数据绑定结果为: "+bindingResult);

	}

}
~~~

其中`LocalValidatorFactoryBean`由之前的`AppConfig`配置类注册







