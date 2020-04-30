---
layout: post
title: 'spring batch手册(一)——配置Step'
subtitle: '配置Step'
date: 2020-04-30
categories: SpringBatch
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: SpringBatch
---

# 1. 配置Step

## 1.2 `TaskletStep`

面向Chunk的处理不是 `Step` 中唯一的处理方式，如果一个 `Step` 必须包括一个简单的存储过程调用该怎么办？你可以像 `ItemReader` 一样实现这个调用并在这个过程完成后返回空值。然而这样做有点不太自然，因为这样就需要有一个没有任何操作的 `ItemWriter` 。Spring Batch 为这个场景提供了 `TaskletStep` 。

`Tasklet` 是一个只有一个 `execute()` 方法的接口，这个方法会被 `Tasklet` 重复调用直到它返回 `RepeatStatus.FINISHED` 或者抛出异常表示失败。每个对 `Tasklet` 的调用都会包含在一个事务中。`Tasklet` 的实现可能会调用一个存储过程，一个脚本或者一个简单的SQL Update语句。

要创建一个 `TaskletStep` ，传递给 `StepBuilder` 的 `tasklet()` 方法的bean必须实现 `Tasklet` 接口，并且在创建 `TaskletStep` 时不能调用 `chunk`  方法。下面是一个简单的tasklet的例子：

```java
@Bean
public Step step1() {
    return this.stepBuilderFactory.get("step1")
                            .tasklet(myTasklet())
                            .build();
}
```

> 如果这个tasklet实现了 `StepListener` 接口， `TaskletStep` 会自动注册这个tasklet作为 `StepListener` 

### 1.2.1 `TaskletAdapter`

就像其他 `ItemReader` 和 `ItemWriter` 接口的适配器一样， `Tasklet` 接口有一个实现 `TaskletAdapter` 允许适配自身到任何已存在的类中。一个非常有用的情形是：已存在一个用来在一些记录中更新一个标签的DAO类，`TaskletAdapter` 可以用来调用这个类，我们甚至不需要为 `Tasklet` 接口写一个适配器，就像下面的例子一样：

```java
@Bean
public MethodInvokingTaskletAdapter myTasklet() {
        MethodInvokingTaskletAdapter adapter = new MethodInvokingTaskletAdapter();

        adapter.setTargetObject(fooDao());
        adapter.setTargetMethod("updateFoo");

        return adapter;
}
```

 ### 1.2.2 Example `Tasklet` Implementation

很多批处理工作包含了很多步骤，这些步骤必须在主要处理程序之前开始来让资源准备就绪或者在程序结束之后清理这些资源。在批处理工作需要处理大量文件的情况下，当文件被成功上传到另一个位置的时候删除本地的特定文件是非常有必要的。下面的例子就是一个 负责上述问题的`Tasklet` 实现。

```java
public class FileDeletingTasklet implements Tasklet, InitializingBean {

    private Resource directory;

    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {
        File dir = directory.getFile();
        Assert.state(dir.isDirectory());

        File[] files = dir.listFiles();
        for (int i = 0; i < files.length; i++) {
            boolean deleted = files[i].delete();
            if (!deleted) {
                throw new UnexpectedJobExecutionException("Could not delete file " +
                                                          files[i].getPath());
            }
        }
        return RepeatStatus.FINISHED;
    }

    public void setDirectoryResource(Resource directory) {
        this.directory = directory;
    }

    public void afterPropertiesSet() throws Exception {
        Assert.notNull(directory, "directory must be set");
    }
}
```

上面的 `Tasklet` 实现删除给定目录的所有文件，需要注意的是，`execute` 方法只会被调用一次，剩下的只有在 `Step` 中引用 `Tasklet` ：

```java
@Bean
public Job taskletJob() {
        return this.jobBuilderFactory.get("taskletJob")
                                .start(deleteFilesInDir())
                                .build();
}

@Bean
public Step deleteFilesInDir() {
        return this.stepBuilderFactory.get("deleteFilesInDir")
                                .tasklet(fileDeletingTasklet())
                                .build();
}

@Bean
public FileDeletingTasklet fileDeletingTasklet() {
        FileDeletingTasklet tasklet = new FileDeletingTasklet();

        tasklet.setDirectoryResource(new FileSystemResource("target/test-outputs/test-dir"));

        return tasklet;
}
```

## 1.3 Controlling Step Flow



### 1.3.1 Sequential Flow 

![Figure 3. Sequential Flow][Sequential Flow]





[Sequential Flow]: data:image/png;base64,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L1687777amuv3n77bZyrqvd47bXXaijchx56qPkORRPqXr169cEHH9xq+Z48eRK6jREffvjhlgp3dHS0KY9D0/5G9Oijj26yfFtbW7/77jvoNlJ88cUXm1n7trS0PPPMM816EJr5990nn3zyb4E7Ojp+/fVX6DZenD17dv/+/dV1X3jhhSY+Ak3+txnBYLBKM3X//ff/+eef0G3UOH/+/N13311pfH799deb++M3/92mXn75ZdHC1Wg0165dg25jx5UrV1Qq1Y2ro48++qjpP/ueuFPc8ePHywq36e8itod0r1+/bjKZ+PKVSCRffvkldJsnPv/8c/70xVNPPbVHPvUeuofn+Ph48S/Uz549C91mizNnzrS1tT333HN75yPvrfvvvvjii+fPn4cuAroI6CKgi4AuAroI6EIXAV0EdBF1o3vhwoV1RCNElf+SKq57/vx5giDab+1A1n8SBHHixIkt6K6trXXcIX04dgxZ/6kYfOTo0aPQhS50oYuELhK6SOgioQtd6EIXutBFQhcJXSR0kdCFLnShi4TuLmaO9kV6HR7VsFtpc1OO+YFA5v/z3Y2zkYPTkf6NaZhdppdy0N1eRleUlMiViEjbvEUAMDDhJghdf2g39iGjIiteyUw+GoNu7XWj0Vc8sjJnorBNtpfdZrd0KarapeqUkyno1la4iS72GOp6phPWaM4aWukxcgdb4TYXdLmjrzNEd1WX6p1ZoedWjP7EoC+i1MqLj0qHI9CtKcMxblC06/lJLhSU5odmokPvtcSy2iEdX0ZSipYPL1oLm9HTXpmCA6BGtL40r6UZYkgtrXTFDBPuzuIbkDq1J/F3urQhLHjcP1V8tHM4Bt3aMtktGAM79WOUa9Ewl2IEh16tKJuQ8wU94LCLDKGeJPuUCiMtORqvXrsaX8q8kDIFkkZ/TKWnNk4Q0N160p4xMQed0hkrGGf7DpUgO4125WiQWSoUd6Hp6XbMax38KzAD0bJ5VKd2+tVmvvqpg0tbnHdJjwVd1baAp72kWNcqtQULwLnSyFmYdwcdDFeLy8VXMBxi/VQzaYGWvNef4/oyObdBZotdFdUzg65q282zaS7e55rq0uuER7bQJPMjLdsz85YdpLyY/BPUeTxOi5riy84y4648OJe+DaqJYP908OBkUOvydet5843zMXQ3maYZr0xLS0miy7VSejwU57k2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

 