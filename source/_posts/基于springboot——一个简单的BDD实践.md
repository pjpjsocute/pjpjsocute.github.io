---
title: springboot+cucumber实践
date: 2023-05-12 15:09:40
categories: 技术
tags:
  - 项目
  - java
comments: true
---

# springboot+cucumber实践

## why BDD

- **满足业务目标。**
- **关注用户需求**
- **良好的可读性**

其实对于我自己来说，也有其他原因：

1.因为客观原因，有时候项目开发结束后才拿到PRD，所以在开发前期，通过一些方式确定明确的业务流程会比直接上手开发可以更容易的发现问题。相比较DD文档，BDD的feature可能是对于非开发人员更易懂的方案。

2.因为文档往往存在滞后，帮助将来的自己或是其他接手的同学去更快的回顾或是了解某个业务的诉求。

<!-- more -->

## 一个样例项目的开始

#### 项目分层：

![WX20230512-151458@2x](基于springboot——一个简单的BDD实践/WX20230512-151458@2x.png)

#### 代码：

以**功能配置**单上线操作为例

application中存在一个上线接口

```java
class interface ConfigurationCmdService{
    /**
     * 上线功能
     *
     * @param cmd
     * @return
     */
    Result<Boolean>  online(ConfigOnlineCmd cmd);
}
/**
* 接口需要实现上线功能
* 假设操作只需要3步：
*	1.查到需要上线的配置
*   2.上线操作
*   3.更新db
*/
class class ConfigurationCmdServiceImpl implements ConfigurationCmdService{
	    
    private final ConfigRepository    repository;


    private final ConfigFactory factory;

    public ConfigurationCmdServiceImpl(ConfigRepository repository,ConfigFactory factory){
        this.repository = repository;
        this.factory = factory;
    }

    public ManualQueryServiceImpl(SnapshotRepository repository, ManualSnapshotFactory factory) {
        this.repository = repository;
        this.factory = factory;
    }
    @AutoWired
    Result<Boolean>  online(ConfigOnlineCmd cmd){
        Config config = repository.queryById(cmd.getId());
        config.online();
        //可能还有其他的一些操作
        return repository.update(config);
    }
}
```

## BDD的接入

### 前置工作

#### cucumber依赖

```xml
<!-- bdd依赖 -->
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-core</artifactId>
</dependency>
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-java</artifactId>
</dependency>
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-junit</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-spring</artifactId>
</dependency>
```

#### 结合junit4

```xml
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven-surefire-plugin.version}</version>
                <configuration>
                    <includes>
                        com.example.**Test.java
                    </includes>
                    <testFailureIgnore>false</testFailureIgnore>
                    <skipTests>false</skipTests>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.apache.maven.surefire</groupId>
                        <artifactId>surefire-junit4</artifactId>
                        <version>2.22.2</version>
                    </dependency>
                </dependencies>
            </plugin>
```

#### 结合jacoco生成单测报告

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>${jacoco.version}</version>
  <executions>
    <execution>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>prepare-package</phase>
      <goals>
        <goal>report</goal>
      </goals>
      <configuration>
        <excludes>
          <!-- -排掉工具类包 比方说，需要排出工具包-->
          <exclude>com.example.util.*</exclude>
        </excludes>
      </configuration>
    </execution>
  </executions>
</plugin>
```



### 第一步

启动类

glue实际上是告诉cucumber启动后去扫描对应包下含有@CucumberContextConfiguration的文件

```java
@RunWith(Cucumber.class)
@CucumberOptions(
        features = {"classpath:feature"},
        glue = {"com.example.step"},
        plugin = {"pretty","html:target/html-reports.htm"}
)
public class ApplicationTest {

}
```

配置在测试中需要启动的bean以及一些需要去mock的bean、

init方法在运行之前触发，reset方法见第四步

```java
@CucumberContextConfiguration
@SpringBootTest(classes = {
        SpringTestConfig.class,
        MockObjectConfiguration.class
    }
)
public class SpringTest {

    @Autowired
    private List<Resetable> resetables;

    @Before
    public void init(){
        CollectionUtils.emptyIfNull(resetables)
                .stream().forEach(v->v.reset());
    }
}
```



### 第二步

在classpath:feature下新建一个feature文件

\#language:zh-CN代表语言为中文

```feature
#language:zh-CN
功能:配置的CMD操作
  场景:上线一条配置
    假设存在以下配置
        |id  | content| status | bizCode |
        |1	 |xxxxx   | AUDIT  | XXXX    |
        |2	 |xxxxx   | DRAFT  | XXXX    |
    当id为"1"上线
        | languageType | bizCode |
        | zh_CN        | 008     |
    那么id为"1"的配置状态为"上线中"
```

### 第三步

实现上述的功能：

```java
public class ContentStep {

    @Autowired
    private FakeConfigRepositoryImpl       configRepository;

    private String                            result;

    @Autowired
    private ConfigurationCmdService cmdService;

    private final String DEFAULT_CODE = "xxx";

    AssertService contentAssertService = new AssertService<>();

    private static final Map<String,String> codeMap = new HashMap<String,String>(){
        {
            put("上线","ONLINE");
            put("审核","AUDIT");
            put("草稿","DRAFT");
        }
    };

    @假如("假设存在以下配置")
    public void 存在以下内容(DataTable dataTable) {
        //根据dataTable去创建一条内容
        List<Config> configs = ConfigTransform.transToConfig(dataTable.entries());
        contentRepository.createAll(configs);
    }

    @那么("id为{string}的配置状态为{string}")
    public void id为的配置状态为(String id,String status){
        Config config = configRepository.queryById(id);
        //判断结果
        Assert.assertEquals(config.getStatus(),codeMap.get(status));
    }

    @当("id为{string}上线")
    public void 在id为的目录下插入一条内容(String id){
        //创建命令
        ContentCreateParam param = createOnlineCmd(id);
        //获得结果
        result = cmdService.online(param).getData().toString();
    }
```

### 第四步

mock db、外部服务。以mock db为例

DB使用一个map来模拟数据库操作

reset操作用于清空map，每一条用例都会自动清空map。

```java
public class FakeConfigRepositoryImpl implements SearchDataRepository ,Resetable{

    @Getter
    private final Map<String, SearchDataDO> doMap;

    private final ConfigConverter converter;

    public FakeSearchDataRepositoryImpl(SearchDataConverter converter) {
        this.doMap = new HashMap<>();
        this.converter = converter;
    }

    /**
     * 创建数据
     *
     * @param config
     * @return
     */
    @Override
    public boolean create(Config config) {
        ConfigDO configDo = converter.convert2DO(config);
        doMap.put(String.valueOf(config.getId()),configDo);
        return true;
    }

    /**
     * 更新数据
     *
     * @param searchData
     * @return
     */
    @Override
    public boolean update(Config config) {
        create(searchData);
        return true;
    }

    @Override
    public void reset() {
        doMap.clear();
    }
```

spring-test和spring-context版本必须一致，否则会报错

