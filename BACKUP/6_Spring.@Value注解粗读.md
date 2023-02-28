# [Spring @Value注解粗读](https://github.com/Winniekun/article/issues/6)

# 概念
1. 创建bean实例，可以使用@Controller、@Service、@Repository、@Component等注解。
2. 依赖注入某个对象，可以使用@Autowired和@Resource注解。
3. 开启事务，可以使用@Transactional注解。
4. 动态读取配置文件中的某个系统属性，可以使用@Value注解。

# 使用
```
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {
	String value();
}
```
从@Value注解的源码，可以看出，其可以标注在字段、方法、参数、注解上，然后是在程序运行期间生效。
## 不通过配置文件注入属性
### 注入普通字符串
```
@Value("normal")
private String normal; // 注入普通字符串
```
### 注入文件资源
```
@Value("classpath:spring/config/config.properties")
private Resource resource; // 注入文件资源
```
### 注入操作系统属性
```
@Value("#{systemProperties['os.name']}")
private String systemPropertiesName; // 注入操作系统的属性
```
### 注入表达式结果
```
@Value("#{T(java.lang.Math).random() * 100.0 }")
private double randomNumber; //注入表达式结果
```
### 注入其他Bean的属性
```
@Value("#{person.name}")
private String name; // 注入其他Bean属性：注入person对象的属性name
```

## 通过配置文件注入属性
```
@Service
public class StudentService {
    @Value("${value.test.userName}")
    private String userName;
    
    public String test() {
        System.out.println(userName);
        return userName;
    }
}
```

```
# username
value.test.userName=kongweikun
```

## 备注
### 乱码问题
```
value.test.userName=\u5f20\u4e09
```
Unicode转义后的字符，转义前是：张三，但是能够正常识别出来：
![image](https://user-images.githubusercontent.com/19886738/221850442-a4e9603e-9a0b-449e-83c8-33d392baee31.png)
```
value.test.userName=张三
```
![image](https://user-images.githubusercontent.com/19886738/221850501-45d009ea-e997-49bc-a754-4017782b3a22.png)
这个的原因是因为在springboot的CharacterReader类中，默认的编码格式是ISO-8859-1，其负责.properties文件中系统属性的读取。该编码格式不支持中文的编码，所以如果系统属性包含中文字符，就会出现乱码。目前主要有三种解决方案：
#### 方案一 手动将ISO-8859-1格式的属性值，转换成UTF-8格式
```
@Service
public class UserService {

    @Value(value = "${value.test.userName}")
    private String userName;

    public String decode() {
        // 转换编码格式
        String userName1 = new String(userName.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8);
        System.out.println(userName1);
        return userName1;
    }
}
```

#### 方案二 将中文字符用unicode编码转义
上述方案只适用于存在极少量的中文配置信息的项目中， 如果项目中包含大量中文系统属性值，每次都需要加这样一段特殊转换代码，会出现大量重复的代码。所以为了避免这种情况，可以直接将系统属性值的中文内容转化为unicode编码（[转义网址](http://www.jsons.cn/unicode/)）。
#### 方案三 使用.yml格式的配置文件
第二种解决方案虽然能够解决中文乱码问题，但是可读性不高，并且需要在配置文件中添加注释。我们知道SpringBoot也支持.yml或.ymal格式的配置，使用.yml或.ymal对系统属性进行配置，就能够杜绝中文乱码问题。因为.yml或.yaml格式的配置文件，最终会使用UnicodeReader类进行解析，它的init方法中，首先读取文件头信息，如果头信息中有UTF8、UTF16BE、UTF16LE编码格式，就采用对应的编码，如果没有，则采用默认UTF8编码。
![image](https://user-images.githubusercontent.com/19886738/221850716-9310cfee-285d-46f9-b8d6-cea179587e4a.png)


### 默认值设置
很多时候使用Java对成员变量设置的默认值，并不能满足我们的日常工作需求。譬如：如果配置了系统属性，userName就用配置的属性值。如果没有配置，则userName用默认值user。
```
@Value(value = "${value.test.userName}")
private String userName = "user";
```
这种方式无法设置默认值，因为设置userName默认值的时间，比@Value注解依赖注入属性值要早，会被直接覆盖。正确的使用@Value设置默认值的方式如下：
```
@Value(value = "${value.test.userName:user}")
private String userName;
```
在需要设置默认值的系统属性名后，加:符号。紧接着，在:右边设置默认值。

### static变量设置值
#### 方案一 set方法手动赋值
虽然静态变量无法使用@Value直接赋值，但是可以通过通过setter方法，然后在setter方法上使用@Value注解给静态变量赋值
```
@Getter
@Component
@Slf4j
public class ImgUtil {
    
    private static String PREFIX;
    
    @Value("image.domain.name.prefix:baidu.com")
    public void setPreFix(String preFix) {
        ImgUtil.PREFIX = preFix;
    }
}
```
#### 方案二 构造方法赋值
```
@Getter
@Component
@Slf4j
public class ImgUtil {
    
    private static String PREFIX;
    
    @Value("image.domain.name.prefix:baidu.com")
    public ImgUtil(String preFix) {
        ImgUtil.PREFIX = preFix;
    }
}
```
## #{} 和 ${}的区别
@Value的能够支持这么多类型的注入，归根揭底就是${...}和#{...}的用法。

  | 功能
-- | --
${...} | 主要用于获取配置文件中的系统属性值，通过:可以设置默认值。如果在配置文件中找不到susan.test.userName的配置，则注入时用默认值。
#{...} | 主要用于通过spring的EL表达式，获取bean的属性，或者调用bean的某个方法。还有调用类的静态常量和静态方法

**${...}**和**#{...}**也可以混合使用，但是在混合使用的时候，${...}需要在#{...}的内层，也就是"#{'${...}'}"的形式，因为Spring执行${...}时机要早于#{...}，当Spring执行外层的${...}时，内部的#{...}为空。
### 其他常见数据类型的注入
#### 集合
对于List类型，除了使用@ConfigurationPropertie定义一个对象之外，也可以直接通过@Value  + Spring EL表达式进行设置
```
@Value("#{'${value.test.list}'.split(',')}")
private List<String> list;
```
```
value.test.list=a,b,c,d
```

#### Map
```
value.test.map={"name":"kongweikun", "age":"18"}
```
```
@Value("#{${value.test.map}}")
private Map<String, String> map;
```
#### 设置默认值
无法通过上述的默认值方式直接设置默认值，需要使用EL表达式的empty方法
```
@Value("#{'${value.test.list:}'.empty ? null : '${value.test.list:}'.split(',')}")
private List<String> list;
```

# 原理
![image](https://user-images.githubusercontent.com/19886738/221851534-29471397-5f9c-48c5-b4d7-b19d489a3f7c.png)


上述就是整个@Value属性填充的时序图。
1. 通过调用**AbstractAutowireCapableBeanFactory**的**populateBean**方法在Bean的初始化阶段填充属性值。
2. **populateBean**方法中调用了**InstantiationAwareBeanPostProcessor**接口的**postProcessProperties**回调方法处理真正的属性填充，
  a. **InstantiationAwareBeanPostProcessor**的实现类有多种，其中的**AutowiredAnnotationBeanPostProcessor**就用来处理属性自动注入的功能。
## Demo
```
@Slf4j
@RestController
public class ValueController {

    @Value("${value.test.age:10}")
    private Integer age;
    
    @Value("${value.test.name:kongweikun}")
    private String name;
}
```
## 第一步：获取被注解的属性（方法）
> 核心逻辑：将对应的Bean中标注为@Value的所有信息封装为InjectionMetadata对象
```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	...
		for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            // 调用postProcessProperties方法处理属性
			PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
			....
		}
	}
    ...
}

public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
	InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
	....
}
```
![image](https://user-images.githubusercontent.com/19886738/221851745-78cbad0f-a2b9-4d87-a0dc-5e59cfb020cf.png)

```
// 解析targetClass的所有属性
ReflectionUtils.doWithLocalFields(targetClass, field -> {
	MergedAnnotation<?> ann = findAutowiredAnnotation(field);
	if (ann != null) {
        // static修饰的属性无法使用@Value注解赋值
		if (Modifier.isStatic(field.getModifiers())) {
			if (logger.isInfoEnabled()) {
				logger.info("Autowired annotation is not supported on static fields: " + field);
			}
			return;
		}
		boolean required = determineRequiredStatus(ann);
		currElements.add(new AutowiredFieldElement(field, required));
	}
});
// 处理方法对应的注解
....
return new InjectionMetadata(clazz, elements);
```
## 第二步：解析
> 通过封装的InjectionMetadata.InjectedElement对象，解析出真正的值，并赋值给当前Bean对应的属性。譬如ValueController.age属性
![image](https://user-images.githubusercontent.com/19886738/221851879-bf2a0eff-c6f2-4eeb-a528-f56753e7708f.png)

```
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	Collection<InjectedElement> checkedElements = this.checkedElements;
	Collection<InjectedElement> elementsToIterate =
			(checkedElements != null ? checkedElements : this.injectedElements);
	if (!elementsToIterate.isEmpty()) {
		for (InjectedElement element : elementsToIterate) {
			element.inject(target, beanName, pvs);
		}
	}
}
```
### Inject方法
```
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	Field field = (Field) this.member;
	Object value;
	if (this.cached) {
		try {
			value = resolvedCachedArgument(beanName, this.cachedFieldValue);
		}
		catch (NoSuchBeanDefinitionException ex) {
			value = resolveFieldValue(field, bean, beanName);
		}
	}
	else {
		value = resolveFieldValue(field, bean, beanName);
	}
    .......
}
```
![image](https://user-images.githubusercontent.com/19886738/221851984-2259105c-fc46-4581-a4da-185316779aa5.png)

### doResolveDependency方法（核心）
> resolveFieldValue方法调用beanFactory.revolveDependency()方法，然后revolveDependency()方法再调 doResolveDependency()方法，执行解析的核心方法
![image](https://user-images.githubusercontent.com/19886738/221852058-5f045153-ba21-4b91-9bed-347aeb4ca553.png)

```
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        // field属性的类型
        Class<?> type = descriptor.getDependencyType();
        // 1. 通过descriptor获取@Value注解的value方法的值    (1)
        Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
        if (value != null) {
            // 如果value是String类型，走这里
            if (value instanceof String) {
                // 2. 开始解析value的值                    (2)
                String strVal = resolveEmbeddedValue((String) value);
                BeanDefinition bd = (beanName != null && containsBean(beanName) ?
                        getMergedBeanDefinition(beanName) : null);
                value = evaluateBeanDefinitionString(strVal, bd);
            }
            TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
            try {
                // 3. value值的类型转换                    (3)
                return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
            }
            catch (UnsupportedOperationException ex) {
                ...
            }
        }
    }finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```
![image](https://user-images.githubusercontent.com/19886738/221852091-323c4d9a-de86-48bc-b278-e149252e5817.png)

```
@Value("${value.test.age:10}")
private Integer age;

第一步：通过descriptor得到${value.test.age:10}
第二步：拆解${value.test.age:10}，得到value.test.age:10，获取值，没有获取到，继续拆解，最后得到值
第三步：此时得到的值是String类型的，需要转换成目标变量声明的类型，此处类型为int
```
#### 第一步
> 通过descriptor解析出注解中value方法中的值 ${key:defalutValue}，解析的入口是getSuggestedValue()，实际执行的方法是findValue()方法
```
protected Object findValue(Annotation[] annotationsToSearch) {
    if (annotationsToSearch.length > 0) {   // qualifier annotations have to be local
        AnnotationAttributes attr = AnnotatedElementUtils.getMergedAnnotationAttributes(
                AnnotatedElementUtils.forAnnotations(annotationsToSearch), this.valueAnnotationType);
        if (attr != null) {
            return extractValue(attr);
        }
    }
    return null;
}
```
通过AnnotatedElementUtils.getMergedAnnotationAttributes()方法得到注解的属性集合，从而获取到@Value注解中value方法中的值。
**这个时候，value的值为${value.test.age:10}**
#### 第二步
> 解析传入的${key:defalutValue}形式字符串，获取到key对应实际的值，解析的入口是resolveEmbeddedValue，实际执行的方法是processProperties()方法
```
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
		final ConfigurablePropertyResolver propertyResolver) throws BeansException {
    // 设置左占位符 ${   
	propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);
    // 设置右占位符  }  
	propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);
    // 设置分隔符  :
	propertyResolver.setValueSeparator(this.valueSeparator);
	StringValueResolver valueResolver = strVal -> {
		String resolved = (this.ignoreUnresolvablePlaceholders ?
				propertyResolver.resolvePlaceholders(strVal) :
				propertyResolver.resolveRequiredPlaceholders(strVal));
		if (this.trimValues) {
			resolved = resolved.trim();
		}
		return (resolved.equals(this.nullValue) ? null : resolved);
	};
```
PropertyPlaceholderHelper.replacePlaceholders()对${key:defalutValue}中，key的值进行替换。
```
private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
    return helper.replacePlaceholders(text, this::getPropertyAsRawString);
}
```

**解析出的为： "10"**
#### 第三步
> 将String类型的值，转化为对应变量声明的数据类型，解析的入口是convertIfNecessary()，实际执行的方法是conversionService.convert(newValue, sourceTypeDesc, typeDescriptor)方法

```
ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
	TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
    // 判断是否支持 从sourceTypeDesc ——> typeDescriptor
	if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
		try {
            // 执行类型转换
			return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
		}
		catch (ConversionFailedException ex) {
			// fallback to default conversion logic below
			conversionAttemptEx = ex;
		}
   }
}
```
![image](https://user-images.githubusercontent.com/19886738/221852441-e6065f25-8a0a-457b-a7fc-0da791c5dfa5.png)

```
@Override
@Nullable
public T convert(String source) {
	if (source.isEmpty()) {
		return null;
	}
	return NumberUtils.parseNumber(source, this.targetType);
}
```
核心方法: **NumberUtils.parseNumber(source, this.targetType)**，转化为出对应的数据类型。
```
public static <T extends Number> T parseNumber(String text, Class<T> targetClass) {
	Assert.notNull(text, "Text must not be null");
	Assert.notNull(targetClass, "Target class must not be null");
	String trimmed = StringUtils.trimAllWhitespace(text);
	...
    // 解析Integer类型数据
	else if (Integer.class == targetClass) {
		return (T) (isHexNumber(trimmed) ? Integer.decode(trimmed) : Integer.valueOf(trimmed));
	}
}
```
### 第三步：赋值
> 还是第二步 inject方法中。
```
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	Field field = (Field) this.member;
	Object value;
	if (this.cached) {
		try {
			value = resolvedCachedArgument(beanName, this.cachedFieldValue);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Unexpected removal of target bean for cached argument -> re-resolve
			value = resolveFieldValue(field, bean, beanName);
		}
	}
	else {
		value = resolveFieldValue(field, bean, beanName);
	}
    // 通过反射，讲获取到的@Value值，赋值给对应的属性
	if (value != null) {
		ReflectionUtils.makeAccessible(field);
		field.set(bean, value);
	}
}
```

# references
* [一文全面深入了解Spring中的@Value注解](https://juejin.cn/post/7113033577138225182)
* [记一次spring注解@Value不生效的深度排查](https://juejin.cn/post/6844904121321930760)
* [细节知多少 - spring @Value注解解析](https://juejin.cn/post/6844903904732266510)
* [【Spring注解驱动开发】如何使用@Value注解为bean的属性赋值](https://www.cnblogs.com/binghe001/p/13216798.html)
* [Spring数据转换](https://cloud.tencent.com/developer/article/1497687)