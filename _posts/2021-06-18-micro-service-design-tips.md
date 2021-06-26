---
layout: post
title: 微服务设计技巧
description: 
date: 2021-06-18
---

# 一文详解微服务架构

[一文详解微服务架构](https://www.zhihu.com/question/65502802)

# 四个技巧

1. 认真设计领域模型及模型间的关系
2. 接口的输入输出只能为基本类型或领域模型
3. 不要依赖其他领域的模型
4. 计算能力封装成SDK

## 认真设计领域模型及模型间的关系

2. 在将需求转化成软件的过程中，首先要作的事情是分析出业务实体及关系，并用类来表示，这些类称之为领域模型。例如下面的考试类、考试学科类

    ``` java
    // 考试
    public class Exam{
        private Integer id;
        private String name;
        private Date startTime;
    }

    // 考试学科
    public class ExamSubject{
        private Integer code;
        private String name;
        private Integer state;
    }
    ```

3. 在设计领域模型时，会受到画面展现的干扰，为其添加一些方便画面展现的属性。也会受到持久化的干扰，为其添加一些方便持久化的属性。这种作法会使领域模型的稳定性差，需要经常变更。

    <img src="../../../assets/images/micro-service-design-tips/examList.png">

    如上图所示，考试列表需要展示考试所包含的学科名称，有可能就会为考试类添加对应的属性subjectNames，很明显subjectNames并不是领域模型的属性，它只是为了满足界面展示的需要而存在的，当界面发生变化时，它也需要跟着改变。

    ``` java
    // 考试
    public class Exam{
        ...

        private String subjectNames;
    }

    ```

4. 领域模型间的关系，传统作法是通过关联另一模型的id来表示。例如：考试学科会属于一个考试

    ``` java
    // 考试学科
    public class ExamSubject{
        private Integer examId;
        ...
    }

    ```

    建议是通过类间的属性关联来表示，例如：

    ``` java
    // 考试
    public class Exam{
        private List<ExamSubject> subjects;
    }

    // 考试学科
    public class ExamSubject{
        private Exam exam;
    }
    ```

    这样组织代码后，数据表示更加完整，参数传递更加方便，可读性也增强。

## 接口的输入输出只能为基本类型或领域模型

``` java
// 考试服务
public interface ExamService{
    void save(Exam exam);
    Exam get(Integer id);
    int getSubjectCnt(Integer id);
    List<ExamSubject> getSubjects(Integer id);
}
```

这样组织代码后，领域服务层的接口签名就非常稳定，不会经常变来变去，可复用性也会增强。

## 不要依赖其他领域的模型

1. 如果某一领域的模型依赖其他领域的模型，两个领域模型相互依赖，不便于单独设计与升级。
2. 让每一个微服务都能高度自治，对外的依赖通过代理模式来解决。

<img src="../../../assets/images/micro-service-design-tips/reportApi.png">

如图所示，报告模块也需要用到考试、试题、题型、学校等对象，全部采用代理的方式，而不会直接依赖其他的领域模型。

## 计算能力封装成SDK

1. 什么是业务能力？对领域模型的CRUD、状态管理、关系维护可以认为是业务能力。
2. 什么是计算能力？跟领域模型没有关系，比如文本处理、图像识别、协议转换等。
3. 单纯的计算能力以SDK的方式进行复用，很方便，没有必要微服务化。
4. 微服务对外提供的业务能力中包含了计算能力，这也是合理的。




# 其他参考资料

1. [什么是Service Mesh？](https://zhuanlan.zhihu.com/p/61901608)
2. [Serverless架构](https://www.jdon.com/48109)





