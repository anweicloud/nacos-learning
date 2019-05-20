# Spring Boot 集成

Nacos Spring Boot Project 有两个部分

- ```nacos-config-spring-boot```

  通过 Nacos Server 和 nacos-config-spring-boot-starter 实现配置的动态变更

- ```nacos-discovery-spring-boot```

  通过 Nacos Server 和 nacos-discovery-spring-b`oot-starter 实现服务的注册与发现。

## 一，启动配置管理

1. 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-config-spring-boot-starter</artifactId>
    <version>${latest.version}</version>
</dependency>
```

2. 配置nacos服务地址`application.yml`

```yml
nacos:
  config:
    server-addr: 127.0.0.1:8848
```

3. 使用 `@NacosPropertySource` 加载 `dataId` 为 `example` 的配置源，并开启自动更新：

   ```java
   @SpringBootApplication
   @NacosPropertySource(dataId = "example", autoRefreshed = true)
   public class NacosConfigApplication {
       public static void main(String[] args) {
           SpringApplication.run(NacosConfigApplication.class, args);
       }
   }
   ```

4. 通过 Nacos 的 `@NacosValue` 注解设置属性值

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

   5. 浏览器访问或使用curl请求config
   
      ```
      http://localhost:8080/config/get
      ```
   
   6. 通过调用 [Nacos Open API](https://nacos.io/zh-cn/docs/open-API.html) 向 Nacos server 发布配置：dataId 为`example`，内容为```useLocalCache=true```
   
      ```bash
      curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=example&group=DEFAULT_GROUP&content=useLocalCache=true"
      ```
   
   7. 再次访问 `http://localhost:8080/config/get`，此时返回内容为`true`，说明程序中的`useLocalCache`值已经被动态更新了。配置成功，这样我们就能动态配置参数了。



## 二，启动服务发现

1. 添加依赖

   ```xml
   <dependency>
       <groupId>com.alibaba.boot</groupId>
       <artifactId>nacos-discovery-spring-boot-starter</artifactId>
       <version>0.2.1</version>
   </dependency>
   ```

2. 配置nacos服务地址`application.yml`

   ```yml
   nacos:
     discovery:
       server-addr: 127.0.0.1:8848
   ```

3. 使用 `@NacosInjected` 注入 Nacos 的 `NamingService` 实例：

   ```java
   @Controller
   @RequestMapping("discovery")
   public class DiscoveryController {
   
       @NacosInjected
       private NamingService namingService;
   
       @RequestMapping(value = "/get", method = GET)
       @ResponseBody
       public List<Instance> get(@RequestParam String serviceName) throws NacosException {
           return namingService.getAllInstances(serviceName);
       }
   }
   
   @SpringBootApplication
   public class NacosDiscoveryApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(NacosDiscoveryApplication.class, args);
       }
   }
   ```

4. 启动 `NacosDiscoveryApplication`，调用 `curl http://localhost:8080/discovery/get?serviceName=example`，此时返回为空 JSON 数组`[]`。因为还没有注册任何服务到nacos。

5. 通过调用 [Nacos Open API](https://nacos.io/zh-cn/docs/open-API.html) 向 Nacos server 注册一个名称为 `example` 服务

   ```bash
   curl -X PUT 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=example&ip=127.0.0.1&port=8080'
   ```

   

6. 再次访问 `curl http://localhost:8080/discovery/get?serviceName=example`，此时返回内容为：

```json
[
  {
    "instanceId": "127.0.0.1-8080-DEFAULT-example",
    "ip": "127.0.0.1",
    "port": 8080,
    "weight": 1.0,
    "healthy": true,
    "cluster": {
      "serviceName": null,
      "name": "",
      "healthChecker": {
        "type": "TCP"
      },
      "defaultPort": 80,
      "defaultCheckPort": 80,
      "useIPPort4Check": true,
      "metadata": {}
    },
    "service": null,
    "metadata": {}
  }
]
```

服务注册成功，那么就可以使用消费者进行调用了，如需要修改配置，使用nacos-config

