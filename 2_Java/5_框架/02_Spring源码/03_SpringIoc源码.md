## 容器持有的类型转换器

`AbstractBeanFactory`持有如下与类型转换相关的字段：

~~~java
private ConversionService conversionService;
private final Set<PropertyEditorRegistrar> propertyEditorRegistrars = new LinkedHashSet<>(4);
private final Map<Class<?>, Class<? extends PropertyEditor>> customEditors = new HashMap<>(4);
private TypeConverter typeConverter;
~~~

### 转换器来源

我们来分析这些类型转换器的非直接手动配置来源：

* `conversionService`：`ApplicationContext`初始化时会检测容器中ID为`conversionService`的ConversionService Bean,将其注册到BeanFactory

* `propertyEditorRegistrars`
  * 通过`CustomEditorConfigurer`这个`BeanFactoryPostProcessor`在容器初始化时向`BeanFactory`注册自定义的`PropertyEditorRegistrar`
  * `ApplicationContext`在初始化`refresh()`时会注册一个`ResourceEditorRegistrar()`
* `customEditors`:通过`CustomEditorConfigurer`这个`BeanFactoryPostProcessor`在容器初始化时向`BeanFactory`注册`PropertyEditor`
* `typeConverter`:`BeanFactory`通过`getTypeConverter()`方法获取`TypeConverter`，如果持有的`typeConverter`为空，那么这个方法会:
  * 实例化一个`SimpleTypeConverter`
  * 将`BeanFactory`持有的`conversionService`注册到`SimpleTypeConverter`
  * 调用将`BeanFactory`持有的`propertyEditorRegistrars`中的`PropertyEditorRegistrar`的`registerCustomEditors()`方法。将`PropertyEditor`注册到`SimpleTypeConverter`
  * 将`BeanFactory`持有的`customEditors`中的注册`PropertyEditor`到`SimpleTypeConverter`

### 初始化`BeanWrapper`

除了`typeConverter`外这些转换器主要会被注册到`BeanWrapper`中，帮助Spring容器在创建Bean实例后设置实例的字段，这个动作在`AbstractBeanFactory.initBeanWrapper()`方法中，其主要逻辑为：

* 将`BeanFactory`持有的 `conversionService`设置给`BeanWrapper`
* 调用`AbstractBeanFactory.registerCustomEditors()`
  * 调用`propertyEditorRegistrars`中中的`PropertyEditorRegistrar`的`registerCustomEditors()`方法。将`PropertyEditor`注册到`BeanWrapper`
  * `customEditors`将`BeanFactory`持有的`customEditors`中的注册`PropertyEditor`到`BeanWrapper`

而关于`TypeConverter`在`BeanFactory`的`getBean()`的最后，`BeanFactory`会尝试用`TypeConverter`将Bean转换为`requiredType`指定的类型

除此之外，`TypeConverter`会在依赖注入、属性赋值等各个方面发挥类型转换的作用。