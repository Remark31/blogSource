---
title: 初识flink
date: 2021-05-15 23:20:53
tags: [flink,实时计算]
---


# 写在前面的话

大部分算法工程都绕不开flink；以前在做搜推的时候，flink更多是用于dump写数据进引擎，即使是处理特征也是相当简单的处理逻辑，对flink了解不多，内心对flink的感知就是... 大概是写一些SQL吧。未来会与flink有更多交集，因此打算借此机会对flink做更多的了解。本文是初识，完全从一个纯新人的角度对flink进行一些总结。


# flink的安装

flink的安装有两种方式，一种是通过源码编译安装，另外一种是Mac系统直接使用brew来进行安装，我这里为了速战速决，实际选用的是后者


> flink的代码库 https://github.com/apache/flink


- 安装方法: brew install apache-flink
- 控制台页面: http://localhost:8081 (P.S. 默认端口为8081，如果被占用，请更换端口)
- 更换端口方法:
    - brew info apache-flink 查看安装路径
    - 在/user/local/Cellar/apache-flink/1.12.2/libexec/conf下，找到flink-conf.yaml文件（熟悉的.yaml文件，想死你了）
    - 修改 rest.port: 8082



# flink的使用

- 启动flink: ./bin/start-cluster.sh
- 停止flink: ./bin/stop-cluster.sh

# hello world

##  源码编写
```java

package demo;

import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.utils.MultipleParameterTool;
import org.apache.flink.util.Collector;
import org.apache.flink.util.Preconditions;

public class SocketTextStreamWordCount {

    private static final String INPUT_STRING = "input";
    private static final String OUTPUT_STRING = "output";

    public static void main(String[] args) throws Exception{
        // 参数获取
        final MultipleParameterTool params = MultipleParameterTool.fromArgs(args);

        // 初始化环境
        final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();


        // 参数网络端口可用
        env.getConfig().setGlobalJobParameters(params);

        // 获取入参请求
        DataSet<String> text = null;

        if(params.has(INPUT_STRING)) {
            for(String input : params.getMultiParameterRequired(INPUT_STRING)) {
                if(text == null){
                    text = env.readTextFile(input);
                } else {
                    text = text.union(env.readTextFile(input));
                }
            }
            Preconditions.checkNotNull(text, "Input DataSet should not be null.");
        } else {
            System.out.println("no input!");
            System.out.println("Use --input to make file input!");
        }



        // 处理数据
        DataSet<Tuple2<String,Integer>> counts = text.flatMap(new LineSplitter()).groupBy(0).sum(1);


        // 结果提交
        if(params.has(OUTPUT_STRING)){
            counts.writeAsCsv(params.get(OUTPUT_STRING), "\n", " ");

            env.execute("WordCount Example");
        } else {
            System.out.println("no output!");
            System.out.println("Use --output to make file output!");
        }


    }


    public static final class LineSplitter implements FlatMapFunction<String,Tuple2<String, Integer>> {

        public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) throws Exception {
            String[] tokens = s.toLowerCase().split(" ");

            for(String token : tokens){
                if(token.length() > 0){
                    collector.collect(new Tuple2<String, Integer>(token , 1));
                }
            }

        }
    }
}

```


## 二方包依赖
```java
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>1.12.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_2.12</artifactId>
            <version>1.12.2</version>
            <scope>provided</scope>
        </dependency>
```


## 打包
直接使用maven打包

```bash
mvn clean package -DskipTests
```

## 提交包并运行

```bash
flink run -c demo.SocketTextStreamWordCount ./target/flink/remark.wordCount-1.0.SNAPSHOT.JAR --input /Users/remark/Learn/flink/test/input/1 --output /User/remark/Learn/flink/test/output/2
```

P.S. 
- 本地提交一定要提交包名(-c)
- 在指定输出和输入文件时一定要使用绝对路径

# flink的基础原理

## 流计算
> 流处理与之前批处理最大的区别在于流是无穷无尽的，以事件为最小原子，随着时间推移逐渐可用。


这里可以先重温一下DDIA的第十一章，
https://github.com/Vonng/ddia/blob/master/ch11.md

https://remark31.github.io/2018/12/28/ddia%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0-%E5%85%AB/#more

## flink 特点

- 具备统一的框架处理有界和无界两种数据流的能力
- 底层支持多种资源调度器，包括Yarn、Kubernetes 等
- 具有极高的可伸缩性
- 极致的流式处理性能，支持本地状态读取，避免了大量网络IO


## flink 基本概念

- Task: 资源调度的最小单位
- TaskSlot:  TaskManager 中的最小资源分配单位，一个 TaskManager 中有多少个 Task Slot 就意味着能支持多少并发的 Task 处理

flink中的两类进程：

- JobManager：协调 Task 的分布式执行，包括调度 Task、协调创 Checkpoint 以及当 Job failover 时协调各个 Task 从 Checkpoint 恢复等
- TaskManager：执行 Dataflow 中的 Tasks，包括内存 Buffer 的分配、Data Stream 的传递等。

![flink](/imgs/flink_basic_content.png)


# 结语

flink的内容较多，接下来的学习思路是先练习，熟悉API，写一些简单的case，再根据case去逐渐了解原理，按照flink的知识大图，大概顺序会是4-1-2-3-8-7

剩下的部分再按需了解，respect!

# 参考文献

- flink的基础概念：https://ververica.cn/developers/flink-basic-tutorial-1-basic-concept/
- flink的搭建：https://ververica.cn/developers/flink-basic-tutorial-1-environmental-construction/
- flink官网：https://ci.apache.org/projects/flink/flink-docs-release-1.13/zh//docs/try-flink/local_installation/
- flink课程：https://github.com/flink-china/flink-training-course#14-datastream-api%E7%BC%96%E7%A8%8B

