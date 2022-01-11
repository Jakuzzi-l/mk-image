# Nacos接入文档（SpringBoot）

## 启动配置管理

### 一、动态修改配置值（注解方式）

#### 1.添加依赖

```xml
<dependency>
    <groupId>com.alibaba.nacos</groupId>
    <artifactId>nacos-spring-context</artifactId>
    <version>${latest.version}</version>
</dependency>
```

#### 2.在`application.properties`中配置*Nacos server*地址

```properties
nacos.config.server-addr=http://192.168.40.237:8848/nacos --测试环境地址
```

如果想指定**命名空间**需添加配置：

> 命名空间：用于进行租户粒度的配置隔离，可用于对不同环境配置的区分隔离，例如开发测试环境和生产环境的资源（如配置、服务）隔离等。

```properties
nacos.config.namespace=b8176606-b502-473c-a18f-a6de8366fb5d --命名空间唯一ID
```

#### 3.动态修改配置值

- 进入搭建好的nacos环境http://192.168.40.237:8848/nacos，登录后，在指定命名空间新建一个dataId,

  group分组可以为默认分组。

  > dataId命名规则（不强制）：项目名称.当前环境对应的profile.具体名称.配置内容的数据格式。
  >
  > 例如：wms.dev.msgCode_en_US.json

- 使用 `@NacosPropertySource` 加载 `dataId` 为 `wms.dev.msgCode_en_US.json` 的配置源，并开启自动更新：

  ```java
  @SpringBootApplication
  @NacosPropertySource(dataId = "wms.dev.msgCode_en_US.json", autoRefreshed = true)
  public class NacosConfigApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(NacosConfigApplication.class, args);
      }
  }
  ```

  -  或者`@NacosPropertySources`

    ```
    @NacosPropertySources({
    	@NacosPropertySource(dataId = "nacos-springboot", autoRefreshed = true),
    	@NacosPropertySource(dataId = "redis", autoRefreshed = true)
    })
    ```

- 通过 Nacos 的 `@NacosValue` 注解设置属性值。

  ```java
  @Controller
  @RequestMapping("config")
  public class ConfigController {
  
      @NacosValue(value = "${useLocalCache:false}", autoRefreshed = true)
      private boolean useLocalCache;
  
      @RequestMapping(value = "/get", method = GET)
      @ResponseBody
      public boolean get() {
          return useLocalCache;
      }
  }
  ```

 通过在配置页面更新dataId指定的内容或者调用 [Nacos Open API](https://nacos.io/zh-cn/docs/open-api.html) 向 Nacos server 发布配置：dataId 为`example`，内容为`useLocalCache=true`来实现动态更新配置

```bash
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=wms.dev.msgCode_en_US.json&group=DEFAULT_GROUP&content=useLocalCache=true"
```

### 二.动态修改配置值（JavaSDK）

#### 1.maven坐标

```xml
<dependency>
    <groupId>com.alibaba.nacos</groupId>
    <artifactId>nacos-client</artifactId>
    <version>${version}</version>
</dependency>
```

请求示例：

```java
private static JSONObject getJsonObject(String dataId, String group) {
        Properties properties = new Properties();
        properties.put(SERVER_ADDR, myConfig.getServerAddr());
        // 配置中心的命名空间id
        properties.put(PropertyKeyConst.NAMESPACE, myConfig.getNamespace());
        ConfigService configService;
        try {
            configService = NacosFactory.createConfigService(properties);
            // 根据dataId、group定位到具体配置文件，获取其内容. 方法中的三个参数分别是: dataId, group, 超时时间
            String content = configService.getConfig(dataId, group, 3000L);
            //配置内容为json，转成json对象
            return JSONUtil.parseObj(content);
        } catch (NacosException e) {
            e.printStackTrace();
        }
        return null;
    }
```

## 启动服务发现

#### 1.添加依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-discovery-spring-boot-starter</artifactId>
    <version>${latest.version}</version>
</dependency>
```

#### 2.在 `application.yml` 中配置 Nacos server 的地址，如果配置了命名空间，则指定命名空间。

```yml
nacos:
  discovery:
    server-addr: 192.168.40.237:8848
    namespace: b8176606-b502-473c-a18f-a6de8366fb5d
```

#### 3.使用 `@NacosInjected` 注入 Nacos 的 `NamingService` 实例

以2B项目为例，注入Namingservice实例，调用注册服务实例方法，参数为`sourceName`:参数名称，`ip`：项目ip地址，`port`:项目端口号。

实现`CommadnLineRunner`或者使用`@PostConstruct`，实现项目启动时，注册服务。

```java
/**
 * @author dengjia
 * @date 2021/6/24
 **/
@SpringBootApplication(scanBasePackages = {"com.heleh.*"})
@MapperScan(basePackages = "com.heleh.*.mapper")
@CrossOrigin
@EnableCaching
public class WmsBoot implements CommandLineRunner {

    @NacosInjected
    private NamingService namingService;
    
    @Value("${server.port}")
    private Integer serverPort;

    public static void main(String[] args) {
        SpringApplication.run(WmsBoot.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        //Nacos注册服务实例
        namingService.registerInstance("WMS", "127.0.0.1", serverPort);
    }
}
```

完整文档请参考[官网](https://nacos.io/zh-cn/docs/quick-start-spring-boot.html)

