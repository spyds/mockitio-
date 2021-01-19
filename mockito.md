# Mockito的使用

## 一、Mockito介绍

​	浅薄地认为Mockito是单元测试的工具。通过它可以模拟程序运行时的所需数据，从而使单元测试运行于理想环境 (区别于实际环境), 让单元测试完全侧重于验证函数逻辑。

## 二、Mockito的使用

**1.引入mockito的依赖**

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>3.2.4</version>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-module-junit4</artifactId>
    <version>2.0.4</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-api-mockito2</artifactId>
    <version>2.0.4</version>
</dependency>
```

**2.编写测试类**

#### **2.1 @InjectMocks、@Spy和@Mock**

**核心代码：**

```java
@RunWith(PowerMockRunner.class)
public class ServiceTest {
    /**
     * 创建一个实例，这个Mock可以调用真实代码的方法
     * 以下使用@Mock或@Spy注解创建的mock将被注入到用该实例中
     */
    @InjectMocks
    private TestService service;
    /**
     * 对函数的调用执行真实逻辑
     * 注意：需要提供实例，否则会报错
     */
    @Spy
    private TestServiceSpyHelper spyHelper = new TestServiceSpyHelper();
    /**
     * 对函数的调用均执行mock，不执行真正部分
     * 没有mock返回结果默认返回为空
     */
    @Mock
    private TestServiceMockHelper mockHelper;

    /**
     * 测试：@spy与@mock的不同
     */
    @Test
    public void testDifferentBetweenSpyAndMock() {
        service.doThingBySpy();
        service.doThingByMock();
    }
}
```

#### **2.2 Mock成员变量的方法**

#####  **(1) mock成员变量的公有方法**

```java
	/**
     * 测试：mock成员变量公有方法
     */
    @Test
    public void testPublic() {
         // 1.没有mock的话，默认返回为空
        System.out.println("result1:" + service.doPublicMethod());

         // 2.使用when(成员变量.方法(参数匹配器)).thenReturn(结果)对公有方法进行mock
        when(mockHelper.doPublicMethod()).thenReturn("mockHelper.doPublicMethod");
        System.out.println("result2:" + service.doPublicMethod());
    }
```

**结果输出：**

```bash
result1:null
result2:mockHelper.doPublicMethod
```



##### **(2) mock成员变量的私有方法**

**测试的目标对象及其包含的成员变量**

```java
@Component
public class TestServiceMockHelper {
    public Integer doPrivateMethod() {
        return privateMethod();
    }
	// 需要被mock的私有方法
    private Integer privateMethod() {
        return 888;
    }
}

@Service
public class TestService {
    @Autowired
    private TestServiceMockHelper mockHelper;

    // 这里需要对mockHelper里的privateMethod进行mock
    public Integer doPrivateMethod() {
        return mockHelper.doPrivateMethod();
    }
}
```

**使用PowerMockito.when(成员变量，方法名，参数匹配器).thenReturn(结果)进行mock**

```java
   /**
     * 测试：mock成员变量的私有方法
     */
    @Test
    public void testPrivateAndStatic() throws Exception {
        PowerMockito.when(mockHelper, "doPrivateMethod").thenReturn(999);
        System.out.println(service.doPrivateMethod());
    }
```

##### (3) mock静态方法

**测试的目标对象**

```java
@Service
public class TestService {
    public Long doStaticMethod(Long lon) {
        // 需要对TestServiceMockHelper的静态方法doStaticMethod进行Mock
        return TestServiceMockHelper.doStaticMethod(lon);
    }
}
```

**Mock静态方法有以下三个步骤**

1. 使用@PrepareForTest注释告诉PowerMockito列出的类将需要在字节码级别上进行操作(调用mockStatic所须需的)
2. 使用 PowerMockito.mockStatic(XXX.class)指示将对哪个类的静态方法进行mock
3. 使用when().then()对静态方法进行mock

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest(TestServiceMockHelper.class)
public class ServiceTest {
    /**
     * 测试：mock静态方法
     */
    @Test
    public void testPrivateAndStatic() {
        PowerMockito.mockStatic(TestServiceMockHelper.class);
        when(TestServiceMockHelper.doStaticMethod(any())).thenReturn(999L);
        System.out.println(service.doStaticMethod(null));
    }
}
```



#### 2.3 构建参数匹配器

​	在对方法进行mock时往往需要根据不同的入参返回不同的mock结果，因此需要不同参数匹配器来满足需求。需要注意的是，如果配置了多个参数匹配器，则会匹配符合匹配条件的最新声明的匹配。构建参数匹配器主要有以下三种方式：

**(1) 模糊匹配**

```java
@Test
public void test() {
    PowerMockito.mockStatic(TestServiceMockHelper.class);
    // 采用any(), anyXXX()等匹配器，要求函数满足任意或指定类型的入参，即可返回mock结果
    when(TestServiceMockHelper.doStaticMethod(anyLong())).thenReturn(999L);
    System.out.println(service.doStaticMethod(1L));
}
```

**(2) 精确匹配**

```java
@Test
public void test() {
    PowerMockito.mockStatic(TestServiceMockHelper.class);
    // 采用eq()匹配器，来严格指定函数入参，只有入参值严格等于指定才返回mock结果
    when(TestServiceMockHelper.doStaticMethod(eq(99L))).thenReturn(999L);
    System.out.println(service.doStaticMethod(99L));
}
```

**(3) 自定义匹配**

当Mockito自带的参数匹配器无法满足需求时(如：要求入参为对象且要求其某些字段等于特定值，才返回指定结果),  此时就需要自定义参数匹配器

```java
@Test
public void test() {
    PowerMockito.mockStatic(TestServiceMockHelper.class);
    // 1.采用ArgumentMatcher<~>构建参数匹配条件，当user.nameCn == 'BEST' 匹配条件生效
    ArgumentMatcher<User> argumentMatcher = (user) -> "BEST".equals(user.getNameCn());
    // 2.采用Mockito.argThat()声明参数匹配器为 argumentMatcher
    when(TestServiceMockHelper.doStaticMethod(Mockito.argThat(argumentMatcher)))
        .thenReturn(false)；
    User user = new User();
    user.setNameCn("BEST");
    System.out.println(service.doStaticMethod(user));
}
```

#### 2.4 验证结果

​	在执行完函数后，需要对测试的结果进行验证，确保函数逻辑的正确性。在日常使用中，验证测试结果多为以下四种场景：

**(1) 验证函数是否抛出异常**

```java
 // 1.使用Mockito自带的验证规则@Rule ExpectedException
@Rule
ExpectedException expectedException = ExpectedException.none();

@Test
public void test() {
    // 2. 验证此声明之后的函数抛出的异常类型均为BizException.class或无异常, 异常信息为"非法数据"
    expectedException.expect(BizException.class);
    expectedException.expectMessage("非法数据");
    TestServiceMockHelper.doStaticMethod(1L);
    // 3. 验证此声明之后的函数抛出的异常类型均为NullPointerException.class或无异常
    expectedException.expect(NullPointerException.class);
    TestServiceMockHelper.doStaticMethod(null);
    // 4. 执行函数无异常，测试结果正确，但由于上一声明生效中，验证结果可能不正确，有可能抛出NullPointerException
    //    因此对于不抛异常结果的验证，建议放置于所有异常声明之前
    TestServiceMockHelper.doStaticMethod(2L);
}
```

**(2) 验证函数执行次数** 

```java
@Test
public void test() {
    service.doPrivateMethod();
    // 使用verify(成员变量, times(执行次数)).方法名()验证service.doPrivateMethod()
    // 执行过程中mockHelper.doPrivateMethod被执行了多少次
    verify(mockHelper, times(1)).doPrivateMethod();
}
```

**(3) 验证函数的返回结果**

```java
@Test
public void test() {
    // 对于有返回结果的函数，直接使用Assert.assertXXX()即可
    Long result = TestServiceMockHelper.doStaticMethod(2L);
    Assert.assertEquals(Long.valueOf(2), result);
}
```

**(4) 验证函数执行过程中的局部变量**

​	在验证测试结果时，往往需要对函数执行过程中的中间变量进行验证(如：验证Dao.insert(Bo)时Bo的值正确性)。此时可以使用ArgumentCaptor<~>与verify()对中间变量进行捕获，从而可以进一步的验证。

```java
@Test
public void test() {
    User user = new User();
    user.setNameCn("Best");
    // 1.调用函数
    service.save(user);
    // 2.构建参数捕获器
    ArgumentCaptor<User> argumentCaptor = ArgumentCaptor.forClass(User.class);
    // 3.调用verify捕获参数
    verify(mockHelper, times(1)).insert(argumentCaptor.capture());
    // 4.获取中间变量
    User userDb = argumentCaptor.getValue();
    System.out.println(userDb);
}
```

