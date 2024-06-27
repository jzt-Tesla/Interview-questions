# 线上JVM内存溢出排查


## 模拟内存溢出

### 设置启动参数

- -Xms100m -Xmx100m -XX:+HeapDumpOnOutOfMemoryError
   - -Xms：堆的最小值
   - -Xmx：堆空间的最大值
   - -XX:+HeapDumpOnOutOfMemoryError	当OOM发生时自动dump堆内存信息
   - -XX:HeapDumpOnOutOfMemoryError=/temp/heapdump.hprof		dump堆内存信息存放目录

### 模拟内存飙升
访问：[http://127.0.0.1:8080/prometheus/test/testUp1](http://127.0.0.1:8080/prometheus/test/testUp1)
   [http://127.0.0.1:8080/prometheus/test/testUp2](http://127.0.0.1:8080/prometheus/test/testUp2)
jps 查看引用进程
jmap -dump:format=b,file=/home/dump.out pid
http://ip:port/actuator/heapdump

### 模拟内存溢出
访问：[http://127.0.0.1:8080/prometheus/test/testOutOfMemory](http://127.0.0.1:8080/prometheus/test/testOutOfMemory)

# 内存溢出问题分析

## 打开mat
![image.png](./img/K0BZDi7fTfDMkF7S/1681456093872-e9ad08e0-cc19-484f-809e-877e061f2007-215303.png)

## 载入dump文件
File - > Open Heap Dump - >选择我们的dump文件(注意文件类型选择All Files或者*.hprof)

## 分析界面

- 在Overview界面点击我们的Dominator Tree 查看内存占用情况（按内存占比从大到小顺序排列）

![企业微信截图_16814563836847.png](./img/K0BZDi7fTfDMkF7S/1681456393896-8bc5914f-9e47-4575-88b0-1448944733af-710505.png)
![企业微信截图_16814564462854.png](./img/K0BZDi7fTfDMkF7S/1681456454012-5e5aea73-2d86-4487-b5c3-8baa34d00a71-264309.png)

-  点击Leak Suspects 可以看到工具帮我们分析出来的问题项 ，点击Details可以查看详情

![企业微信截图_16814565888698.png](./img/K0BZDi7fTfDMkF7S/1681456596829-32f4db94-b2b4-49a8-b436-0c55d16066aa-594294.png)
![企业微信截图_16814566379461.png](./img/K0BZDi7fTfDMkF7S/1681456644091-d10795cc-8925-4558-8b0a-11ccf0e7095c-949319.png)

## SpingBoot项目搭建

## 添加maven依赖
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

## 配置application.properties
```properties
server.port=8080
spring.application.name=prometheus-jvm
management.endpoints.web.exposure.include=*
management.metrics.tags.application=${spring.application.name}
```

## 添加监控JVM的配置类
```java
@Bean
public MeterRegistryCustomizer<MeterRegistry> configurer(@Value("${spring.application.name}") String applicationName){
    return registry -> registry.config().commonTags("application", applicationName);
}
```

## 启动SpringBoot项目
启动成功后，访问[http://127.0.0.1:8080/actuator/prometheus](http://127.0.0.1:8080/actuator/prometheus)查看指标是否正常

# Prometheus搭建

## 下载地址
[https://prometheus.io/download/](https://prometheus.io/download/)

## prometheus.yml添加配置
```shell
#名称随便取
 - job_name: "prometheus-jvm"
    # 多久采集一次数据
    scrape_interval: 5s
    # 采集时的超时时间
    scrape_timeout: 5s
    # 采集的路径
    metrics_path: '/actuator/prometheus'
    # 采集SpringBoot的服务地址
    static_configs:
       - targets: ['127.0.0.1:8080']
```
最终是这个样子
![企业微信截图_16813045933500.png](./img/K0BZDi7fTfDMkF7S/1681304602409-f880d873-4877-4579-937f-6cf4c667c322-517128.png)

## windows下启动 prometheus
在prometheus文件目录下执行，**prometheus.exe --config.file=prometheus.yml**
![企业微信截图_16813060957760.png](./img/K0BZDi7fTfDMkF7S/1681306101248-85ce5793-ba83-4a67-b2e6-84a435e19a52-762833.png)
访问：[http://127.0.0.1:9090/targets](http://127.0.0.1:9090/targets)

# Grafana搭建

## 下载地址
[https://grafana.com/grafana/download?pg=get&platform=windows&plcmt=selfmanaged-box1-cta1](https://grafana.com/grafana/download?pg=get&platform=windows&plcmt=selfmanaged-box1-cta1)

## 启动

- 解压完直接进入bind目录，双击grafana-server.exe启动服务
- 访问：[http://127.0.0.1:3000/](http://127.0.0.1:3000/)
- 用户名密码随便输入，自己记住

## 配置模板

- 模板库地址：[https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)
- 视频模板采用地址：[https://grafana.com/grafana/dashboards/4701-jvm-micrometer/](https://grafana.com/grafana/dashboards/4701-jvm-micrometer/)

### 配置prometheus数据源
点击Prometheus 
![image.png](./img/K0BZDi7fTfDMkF7S/1681307820503-082bf001-e7ca-4947-a0bb-3cc3ed60f8ce-128075.png)
![image.png](./img/K0BZDi7fTfDMkF7S/1681308595265-a119a08f-0409-4adc-b9a1-37d7c0f8c9d8-185175.png)

### 导入模板
![image.png](./img/K0BZDi7fTfDMkF7S/1681307368455-71134c4b-17aa-4d73-8465-c19b69ff0425-872787.png)
![image.png](./img/K0BZDi7fTfDMkF7S/1681308455241-0d630026-6994-4d28-9150-f0be35bcd463-007752.png)

### 




> 原文: <https://www.yuque.com/tulingzhouyu/sfx8p0/qdh0ck3zzgqew1b3>