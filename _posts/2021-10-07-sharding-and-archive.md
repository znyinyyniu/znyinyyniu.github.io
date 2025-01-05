---
layout: post
title: 分库分表和数据归档
description: 
date: 2021-10-07
---

# ShardingSphere HelloWorld

* 两个数据库db1、db2，都有一张表ei_exam，当exam_id为基数时落在db1，当exam_id为偶数时落在db2。

* 获取DataSource

``` java
public static final DataSource getDataSource() throws SQLException {
    Map<String, DataSource> dataSourceMap = createDataSourceMap();

    ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();

    // ......

    return ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig,
            new HashMap<>(16), new Properties());
}
```

* 初始化物理数据源map

``` java
private static Map<String, DataSource> createDataSourceMap() throws SQLException {
    Map<String, DataSource> dataSourceMap = new ConcurrentHashMap<>();

    for (int i = 0; i < 2; i++) {
        String dbName = "db"+(i+1);
        String url = "jdbc:mysql://10.8.0.246:3306/db"+(i+1);
        String userName = "user";
        String password = "psw";
        dataSourceMap.put(dbName, initDataSource(url, userName, password));
    }

    return dataSourceMap;
}

private static DataSource initDataSource(String url, String userName, String password) throws SQLException {
    DruidDataSource result = new DruidDataSource();
    result.setDriverClassName(com.mysql.jdbc.Driver.class.getName());
    result.setUrl(url);
    result.setUsername(userName);
    result.setPassword(password);
    result.setFilters("stat");

    result.setMaxActive(5);
    result.setInitialSize(2);
    result.setMinIdle(2);

    result.setMaxWait(6000);
    result.setTimeBetweenEvictionRunsMillis(60000);
    result.setMinEvictableIdleTimeMillis(300000);

    result.setTestWhileIdle(true);
    result.setTestOnBorrow(false);
    result.setTestOnReturn(false);

    result.setPoolPreparedStatements(true);
    result.setMaxOpenPreparedStatements(20);

    result.setAsyncInit(true);

    return result;
}
```

* 定义分库分表规则配置

``` java
ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();

//shardingRuleConfig.setDefaultDataSourceName(DataSourceUtil.getDefaultDataSourceName());

List<String> tableList = new ArrayList<>();
tableList.add("ei_exam");
shardingRuleConfig.getBindingTableGroups().add(StringUtils.join(tableList, ","));
for (String table : tableList) {
    TableRuleConfiguration tableRuleConfiguration = new TableRuleConfiguration();
    tableRuleConfiguration.setLogicTable(table);
    shardingRuleConfig.getTableRuleConfigs().add(tableRuleConfiguration);
}

shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(
        new ComplexShardingStrategyConfiguration("exam_id", new DatabaseShardingAlgorithm()));



public class DatabaseShardingAlgorithm  implements ComplexKeysShardingAlgorithm {

    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, Collection<ShardingValue> shardingValues) {
        Collection<String> dbNames = new ArrayList<>();
        Integer examId = null;

        Collection<Integer> examIds = getShardingValue(shardingValues, "exam_id");
        for (Integer examId1 : examIds) {
            examId = examId1;
            break;
        }

        if(examId % 2 == 0){
            dbNames.add("db2");
        }else{
            dbNames.add("db1");
        }

        return dbNames;
    }

    private Collection<Integer> getShardingValue(Collection<ShardingValue> shardingValues, final String key) {
        Collection<Integer> valueSet = new ArrayList<>();
        Iterator<ShardingValue> iterator = shardingValues.iterator();
        while (iterator.hasNext()) {
            ShardingValue next = iterator.next();
            if (next instanceof ListShardingValue) {
                ListShardingValue value = (ListShardingValue) next;
                if (value.getColumnName().equals(key)) {
                    return value.getValues();
                }
            }
        }
        return valueSet;
    }
}
```

* 数据源更换为ShardingDataSource后，所有的分库分表逻辑都交由ShardingSphere处理了，对用户完全透明

# 背景

1. mysql数据库、单表记录超过7000万（机械硬盘）或单表记录超过1亿（SSD）、数据库查询性能明显下降
2. 一场考试5000个学生，20题，打分记录为10万。mysql单表最高支持1000场考试
3. 当前，一次期末考试，考试科次能达到4000场左右，打分记录为4亿，所以mysql单表是无法支撑的，需要进行分库处理

<div style="padding:10px;">
    <img src="../../../assets/images/sharding-and-archive/marking-ER-diagram.png">
</div>

# 分库方案

1. 分成4个库，两个mysql实例承载，这种规模是无法承接两次大考的。
2. 考试数据有个特点，最近一个月的数据访问会很频繁（热数据），一个月前的数据很少访问（冷数据）
3. 因此，以自然年为维度，每年8个库，4个库用于存放热数据，4个库用于存放冷数据，有一个归档机制，每天定时对一个月前的数据进行归档
4. 定期（每年、每个学期、市场有重大拓展时）对业务规模进行评估，根据业务规模对分库个数进行扩容。
5. 分库扩容后，最好不要进行数据迁移，简化扩容过程

# 分库实现要点

* 新建考试时，为每个学科指定数据源名称，并记录到一个数据库表中

``` java
/*
    * 新建考试
    */
examId = examDao.insert(exam);

for (String subjectId : subjects) {
    shardingService.route(examId, Integer.parseInt(subjectId));
}

exam.setId(examId);

examSessionDao.insert(exam, subjects);
examDao.insertRooms(exam, roomList);
```

* 分库算法中，从数据表中获取某场考试的数据源名称。数据源名称一旦指定，几乎不会变化，所以采用缓存以提升性能。

``` java
String dbName = RouteCacheUtil.getInstance().getDBName(examId, subjectID);


public String getDBName(Integer examId, Integer subjectId) {
    String key = String.format("sharding:exam:dbName:%d:%d", examId, subjectId);
    try {
        return this.cache4DBName.get(key, () -> {
            String dbName = shardingService.getDBName(examId,subjectId);
            if (dbName == null) {
                return "";
            }
            return dbName;
        });
    } catch (ExecutionException e) {
        throw new CFSysException("cache4DBName.get异常。", e);
    }
}
```

* 采用这种静态路由策略，分库扩容时，不需要进行数据迁移，非常简单

# 归档实现要点

* 每天凌晨2:30开始进行数据归档，为避免归档业务对正常业务的影响，至凌晨5:30，要停止归档任务的运行

``` java
SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
Date t1 = sdf.parse(sdf.format(new Date()));
Date t2 = sdf.parse(EtcdUtil.getV("/job/marking-archive-job/archiveEndTime", ""));
if (t1.getTime() > t2.getTime()) {
    logger.warn("数据归档已暂停");
    break;
}
```

* 数据归档过程中，若出现失败，则该场考试归档结束。该场考试数据可正常使用，也可以再次重新归档

``` java
// 数据进入归档库，采用先删除后插入的策略
archiveService.archiveScanData(examId, subjectId);
archiveService.archiveMarkingData(examId, subjectId);
archiveService.archiveOnlineData(examId, subjectId);

// 归档库插入完成后，原库数据删除采用事务控制
deleteDataService.deleteData(examId, subjectId);

// 归档失败后，清缓存、更改状态
archiveService.archiveFail(examId, subjectId);
```

* 数据归档过程中，该场考试的数据不允许操作

``` java
// 获取某场考试对应分库数据源名称时，进行归档状态检测
if (RouteCacheUtil.getInstance().getIsArchiving(examId, subjectId)) {
    String msg = "数据正在归档中，大约5至20分钟后恢复。";
    throw new CFBizException("888888", msg);
}
```

* 数据归档完成后，要清空各dubbo服务部署结点本地的路由缓存

``` java
public void archiveDone(Integer examId, Integer subjectId) {
    ArchiveRecord archiveRecord = new ArchiveRecord();
    archiveRecord.setExamId(examId);
    archiveRecord.setSubjectId(subjectId);
    archiveRecord.setEndTime(new Date());
    archiveRecord.setArchiveState(ArchiveState.success);
    archiveRecordDao.update(archiveRecord);

    shardingService.route2Archive(examId, subjectId);

    // 数据库中写入清空分库路由缓存的消息
    ClearShardingRouteCacheMsg msg = new ClearShardingRouteCacheMsg();
    msg.setCreateTime(new Date());
    msg.setExamId(examId);
    msg.setSubjectId(subjectId);
    clearShardingRouteCacheMsgDao.insert(msg);
}

// dubbo服务中使用定时任务获取消息并清空本地缓存
private static void schedule() throws SchedulerException {
    SchedulerFactory sf = new StdSchedulerFactory();
    Scheduler sched = sf.getScheduler();

    JobDetail job = newJob(ClearShardingRouteCacheJob.class).withIdentity("clearShardingRouteCacheJob", "group1").build();

    String cronExpression = "0/10 * * * * ?";
    CronTrigger trigger = newTrigger().withIdentity("clearShardingRouteCacheJobTrigger", "group1")
            .withSchedule(cronSchedule(cronExpression)).build();

    sched.scheduleJob(job, trigger);

    sched.start();
}
```

# JDBC操作数据库的基本步骤

``` java
// 1、获取数据库连接  
conn = datasource.getConnection();  
// 2、获取数据库操作对象  
stmt = conn.createStatement();  
// 3、定义操作的SQL语句  
String sql = "select * from user where id = 100";  
// 4、执行数据库操作  
rs = stmt.executeQuery(sql);  
// 5、获取并操作结果集  
while (rs.next()) {  
    System.out.println(rs.getInt("id"));  
    System.out.println(rs.getString("name"));  
}  
```

# ShardingSphere源码剖析

1. ShardingDataSource类

        实现DataSource接口
        管理物理数据源、分库分表规则
        重写getConnection方法，返回ShardingConnection实例

2. ShardingConnection类

        重写prepareStatement方法，返回ShardingPreparedStatement实例

3. ShardingPreparedStatement类

        重写executeQuery方法
            sqlRoute();
            initPreparedStatementExecutor();
            preparedStatementExecutor.executeQuery()
                获取物理数据源
                JDBC操作数据库的基本步骤

# ShardingSphere内核剖析

[内核剖析](https://shardingsphere.apache.org/document/legacy/3.x/document/cn/features/sharding/principle/)

# ShardingSphere单元测试剖析

* @RunWith(Parameterized.class)，参数化的形式批量生成单元测试类

``` java
public class ParametersDemo {
    private Integer examId;
    private String examName;

    public ParametersDemo(Integer examId,String examName){
        this.examId=examId;
        this.examName=examName;
    }

    @Parameters(name="{0} -> {1}")
    public static Collection<Object[]> getTestParameters(){
        List<Object[]> parameters=new ArrayList<>();

        Object[] a=new Object[2];
        a[0]=1;
        a[1]="exam"+1;
        parameters.add(a);

        Object[] b=new Object[2];
        b[0]=2;
        b[1]="exam"+2;
        parameters.add(b);

        return parameters;
    }

    @Test
    public void test1(){
        Assert.assertEquals("exam"+this.examId,this.examName);
    }

    @Test
    public void test2(){
        Assert.assertEquals("aaa"+this.examId,this.examName);
    }
}
```

* IntegrateSupportedSQLParsingTest类，分别从两种xml文件中加载要进行解析单元测试的sql语句及解析后的比对结果，通过参数化的方式进行批量单元测试。

* 分析sql解析中词法分析的单元测试代码


# 参考资料

[ShardingSphere](https://shardingsphere.apache.org/document/legacy/3.x/document/cn/overview/)