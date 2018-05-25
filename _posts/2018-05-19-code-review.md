---
layout: post
title: 代码检查
description: 
date: 2018-05-19
---

# 组织形式

## 完全的会议形式

召开一个代码检查会议，代码作者向与会者讲解代码，与会者首先要理解代码、然后要发现代码中的问题、然后要就问题进行讨论并达成一致意见。这种组织形式有很多缺点：
1. 非常耗时。
2. 与会者越多，讨论越多，意见难统一，导致成本增大。
3. 有一些与会者不了解需求，无法跟上节奏，白白浪费时间。
4. 与会者无法集中精力，导致无法发现代码中真正的问题，最后也只能就一些代码风格及规范进行讨论。

## 线下检查+会议讨论

由经验丰富的人员独自进行代码检查，并将发现的问题记录下来。然后再召开一个代码检查会议，与会者就发现的问题进行讨论并达成一致意见。然后代码作者对相关问题进行修改。然后再由原代码检查者进行修改确认，达到闭环。这种组织形式能解决很多问题：
1. 代码检查者可以灵活的选择代码检查的时间，检查过程中不容易被干扰，这样就能集中精力，发现代码中真正的问题。
2. 由于会议上只需就发现的问题进行讨论并达成一致意见，所以会议非常高效。
3. 由于代码检查者在线下作了充分的准备，会议上能让所有与会者快速的理解代码的问题，学到知识。

# 小技巧

1. 代码检查者要由经验较丰富的人来承担。
2. 新加入成员的代码要加强检查。
3. 经验欠丰富成员的代码要加强检查。
4. 使用diff工具快速定位代码的变化。
5. 经常进行代码检查, 不要攒了1w行才进行。

# 关注点

## 体系结构和代码设计

1. 新代码与全局的架构是否保持一致？
2. 新代码是否参照了因不规范而淘汰的旧代码？
3. 代码的位置是否正确？
4. 代码是否被过度设计了？
5. 代码是否存在复用问题？
6. 异常处理是否正确？

## 可读性和可维护性

1. 字段、变量、参数、方法、类的命名是否真实反映它们所代表的事物, 是否能够望文生义?

    《阿里巴巴java开发手册》

    [Angular语法、约定和应用组织结构的官方指南](https://www.angular.cn/guide/styleguide)

2. 所采用的术语、词汇是否一致？

## 安全

1. 是否新的路径和服务需要认证？
2. 是否存在sql注入攻击？
3. 数据是否需要加密？
4. 密码是否被很好地控制？
5. 代码的运行是否应该被日志记录或监控？是否正确地使用？

## 资源释放

1. 是否存在内存泄漏?
2. 是否存在内存无限增长? 例如, 如果审查者看到不断有变量被追加到list或map中, 那么就要考虑下这个list或map什么时候失效, 或清除无用数据
3. 代码是否及时关闭了连接或数据流?
4. 资源池配置是否是否正确? 有没有过大或者过小?

## 异常情况处理

1. 超时是否能够正确处理?
2. 调用接口出错的时候, 是否有出错处理逻辑, 并且处理正确?
3. 进程意外重启后, 是否能够恢复到崩溃前的环境?

## 多线程环境的正确性

1. 代码是否使用了正确的适合多线程的数据结构
2. 代码是否存在竞态条件（race conditions）？多线程环境中代码非常容易造成不明显的竞态条件。作为审查者，可以查看不是原子操作的get和set
3. 代码是否正确使用锁？和竞态条件相关，作为审查者你应该检查被审代码是否允许多个线程修改变量导致程序崩溃。代码可能需要同步、锁、原子变量来对代码块进行控制

## 代码级优化

1. 代码是否在不需要的地方使用同步或锁操作？如果代码始终运行在单线程中，锁往往是不必要的
2. 代码是否可以使用原子变量替代锁或同步操作？
3. 代码是否使用了不必要的线程安全的数据结构？比如是否可以使用ArrayList替代Vector？
4. 代码是否在通用的操作中使用了低性能的数据结构？如在经常需要查找某个特定元素的地方使用链表
5. 代码是否可以使用批量接口并从中获得性能提升？
6. 条件判断语句或其他逻辑是否可以将最高效的求值语句放在前面来使其他语句短路？
7. 代码是否存在许多字符串拼接，而没有使用StringBuilder或StringBuffer？

## 其他

1. 虽然缓存是一种能防止过多高消耗请求的方式，但其本身也存在一些挑战。如果审查的代码使用了缓存，你应该关注一些常见的问题，如，不正确的缓存失效方式。
2. 是否需要事务保证？是否可以保证最终一致性，业务是否可以接受？

# 常见问题举例

## 异常处理

1. 异常链丢失
2. 异常信息不记录到日志文件
3. 没有必要的异常封装
4. job定时任务中的异常未处理
5. 线程异步执行过程中的异常未处理
6. 吃掉异常

<script src="https://gist.github.com/znyinyyniu/5d407887be23571fb18be3f72d7652e8.js"></script>

## 枚举

1. 字符串转枚举应直接使用valueOf方法
2. code、value等转枚举的方法应直接定义在对应的枚举类中，以便将相关联的代码进行高内聚，从而提升可维护性

    <script src="https://gist.github.com/znyinyyniu/fc148bf0e7628c10fe892f1cede0b2b9.js"></script>

3. angular组件的html模板中应直接使用枚举，而不是写死字符串

    <script src="https://gist.github.com/znyinyyniu/f4c17c8a5398902b83b66a8a0c936dfd.js"></script>

4. TypeScript开发过程中，如果有枚举，一定要用，而不是写死字符串

    <script src="https://gist.github.com/znyinyyniu/c525d5e574e125f1a01f99ba1dcf3a7e.js"></script>
    <script src="https://gist.github.com/znyinyyniu/3dbcd97137d40c5ca0495f8220d6ae0c.js"></script>

## 方法命名存在不必要的重复

<script src="https://gist.github.com/znyinyyniu/347f5ee37af9cb8d2e0790843aec15b4.js"></script>
<script src="https://gist.github.com/znyinyyniu/81fbf1ca88ed4cfbe2ab66a5a987c707.js"></script>

## dao类中方法名的前缀

dao类中方法命名可以使用固定前缀：find、insert、del、delete、update、upsert。

<img src="../../../assets/images/code-review/dao.png">

## sql语句拼接注意事项

1. 使用StringBuffer或StringBuilder来拼接语句
2. 以sql关键字作为换行的依据， 每行以空格开头。
3. 不要使用select *,每行只写五个字段名， 超过五个后放到下一行， 且以tab开头。

<script src="https://gist.github.com/znyinyyniu/0f990c003cd31b369baaa2fb372dc055.js"></script>

## 以卫语句取代嵌套条件表达式

需求：验证qq是否合法：5-15位，不能以0开头，全是数字

<script src="https://gist.github.com/znyinyyniu/e064f25d771bd0d4aa67e9d08845f253.js"></script>
<script src="https://gist.github.com/znyinyyniu/bc0ef796cd0bc049731b3cd1e9f762db.js"></script>

## 流水帐式的编码风格

1. 将代码组织成流水帐式的风格， 从而提升可读性与可维护性
2. 什么是流水帐式的风格呢？ 意思就是一件事一件事的完成， 同样的事不要重复。 这样就能避免代码的层次嵌套， 提升可读性； 也能避免代码重复， 提升可维护性。

<script src="https://gist.github.com/znyinyyniu/d7e5ef3a8cdba68a85936d627824bcbd.js"></script>

代码流水帐为：
1. 处理参数。
2. 如果status=1， 获取数据， 生成resultMap。 否则， 以另一种方式获取数据， 生成resultMap。

生成resultMap这件事被条件分支进行了重复， 代码可读性差， 代码不便于修改。

<script src="https://gist.github.com/znyinyyniu/9238d15df784d82f7023e178b77e2582.js"></script>

## PageTip应放在路由组件

1. 建议PageTip组件放在路由组件中，不要放置在内部组件中
2. 内部组件对外暴露三个事件（onScuccess、onWarn、onError），以向外传递组件内部的信息
3. 这样就可以避免出现很多个PageTip组件实例

<script src="https://gist.github.com/znyinyyniu/0adaaf9d6aff538998213612c50765c0.js"></script>

## 尽量使用Apache Commons相关工具类

1. Apache Commons Lang、Apacche Commons IO中提供了很多实用方便的工具类，应尽量使用
2. 比较常用的有StringUtils、HashCodeBuilder、IOUtils、FileUtils等

<script src="https://gist.github.com/znyinyyniu/9bbd07ba8c5d4576ff6ab492fc4809c2.js"></script>

# 参考资料

1. 《重构 改善既有代码的设计》
