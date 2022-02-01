+++
title = "ContiPerf::优雅且方便的单元压力测试工具"
date = "2018-07-22 14:12:00"
url = "archives/11"
tags = ["ContiPerf","性能测试","JUnit","单元测试"]
categories = ["测试"]
+++

`ContiPerf` 是一个轻量级的单元测试工具，基于`JUnit 4`二次开发，使用它基于注解的方式，快速在本地进行单元压测并提供详细的报告。

### 示例 ###

#### 新建 SpringBoot 工程 ####

POM文件中的核心依赖如下：  

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.databene</groupId>
    <artifactId>contiperf</artifactId>
    <version>2.1.0</version>
    <scope>test</scope>
</dependency>
```

#### 测试接口以及实现 ####

```java
package com.wuwenze.contiperf.service;

import java.util.List;

public interface ContiperfExampleService {

    List<String> findAll();
}
```

```java
import com.wuwenze.contiperf.service.ContiperfExampleService;

import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class ContiperfExampleServiceImpl implements ContiperfExampleService {
    private final Random RANDOM = new Random();

    @Override
    public List<String> findAll() {
        try {
            int sleepSecond = RANDOM.nextInt(10);
            log.info("#findAll(): sleep {} seconds..", sleepSecond);
            Thread.sleep(sleepSecond * 1000);
        } catch (InterruptedException e) {
            // ignore
        }
        List<String> resultList = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            resultList.add("string_" + i);
        }
        return resultList;
    }
}
```

#### 构建单元测试 ####

```java
package com.wuwenze.contiperf.service;

import com.wuwenze.contiperf.ContiperfExamplesApplication;

import org.databene.contiperf.PerfTest;
import org.databene.contiperf.junit.ContiPerfRule;
import org.junit.Rule;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;


@RunWith(SpringRunner.class)
@SpringBootTest(classes = ContiperfExamplesApplication.class)
public class ContiperfExampleServiceTest {
    @Rule
    public ContiPerfRule i = new ContiPerfRule();

    @Autowired
    private ContiperfExampleService contiperfExampleService;

    @Test
    @PerfTest(threads = 1000, duration = 1500)
    public void findAll() {
        contiperfExampleService
                .findAll()
                .forEach(System.out::println);
    }
}
```

#### 最终执行效果 ####

![acb82b06-1414-48f3-ad75-1bc7280f8ded.png][]

#### 查看测试报告 ####

![6468a841-88b5-4654-9579-18c609348c3e.png][]

### 总结 ###

#### PerfTest参数 ####

| 参数                           | 描述                           |
|------------------------------|------------------------------|
| @PerfTest(invocations = 300) | 执行300次，和线程数量无关，默认值为1，表示执行1次； |
| @PerfTest(threads = 30)      | 并发执行30个线程，默认值为1个线程；          |
| @PerfTest(duration = 20000)  | 重复地执行测试至少执行20s。              |


以上三个属性可以组合使用，其中Threads必须和其他两个属性组合才能生效。当Invocations和Duration都有指定时，以执行次数多的为准。

```bash
// 如果执行方法300次的时候执行时间还没到100ms，则继续执行到满足执行时间等于100ms，如果执行到50次的时候已经100ms了，则会继续执行之100次。
@PerfTest(invocations = 300, threads = 2, duration = 100)，

// 如果你不想让测试连续不间断的跑完，可以通过注释设置等待时间，每执行完一次会等待30~80ms然后才会执行下一次调用。
@PerfTest(invocations = 1000, threads = 10, timer = RandomTimer.class, timerParams = { 30, 80 })

// 在开多线程进行并发压测的时候，如果一下子达到最大进程数有些系统可能会受不了，ContiPerf还提供了“预热”功能
@PerfTest(threads = 10, duration = 60000, rampUp = 1000);

// 启动时会先起一个线程，然后每个1000ms起一线程，到9000ms时10个线程同时执行，那么这个测试实际执行了69s，如果只想衡量全力压测的结果，那么可以在注释中加入warmUp，即
@PerfTest(threads = 10, duration = 60000, rampUp = 1000, warmUp = 9000);
// 那么统计结果的时候会去掉预热的9s。
```

#### Required参数 ####

| 参数                                       | 描述                               |
|------------------------------------------|----------------------------------|
| @Required(throughput = 20)               | 要求每秒至少执行20个测试；                   |
| @Required(average = 50)                  | 要求平均执行时间不超过50ms；                 |
| @Required(median = 45)                   | 要求所有执行的50%不超过45ms；               |
| @Required(max = 2000)                    | 要求没有测试超过2s；                      |
| @Required(totalTime = 5000)              | 要求总的执行时间不超过5s；                   |
| @Required(percentile90 = 3000)           | 要求90%的测试不超过3s；                   |
| @Required(percentile95 = 5000)           | 要求95%的测试不超过5s；                   |
| @Required(percentile99 = 10000)          | 要求99%的测试不超过10s;                  |
| @Required(percentiles = "66:200,96:500") | 要求66%的测试不超过200ms，96%的测试不超过500ms。 |


#### 测试报告 ####

最终的测试报告位于`target/contiperf-report/index.html`，使用浏览器打开即可。


[acb82b06-1414-48f3-ad75-1bc7280f8ded.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180722/acb82b06-1414-48f3-ad75-1bc7280f8ded.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[6468a841-88b5-4654-9579-18c609348c3e.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180722/6468a841-88b5-4654-9579-18c609348c3e.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg