---
layout: post
title: 分库分表和数据归档
description: 
date: 2021-10-07
---

# 背景

1. mysql数据库、单表记录超过7000万（机械硬盘）或单表记录超过1亿（SSD）、数据库查询性能明显下降
2. 一场考试5000个学生，20题，打分记录为10万。mysql单表最高支持1000场考试
3. 当前，一次期末考试，考试科次能达到4000场左右，打分记录为4亿，所以mysql单表是无法支撑的，需要进行分库处理

# ShardingSphere HelloWorld

1. 两个数据库db1、db2，都有一张表ei_exam，当exam_id为基数时落在db1，当exam_id为偶数时落在db2。

2. 获取DataSource

``` java
public static final DataSource getDataSource() throws SQLException {
    Map<String, DataSource> dataSourceMap = createDataSourceMap();

    ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();

    // ......

    return ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig,
            new HashMap<>(16), new Properties());
}
```

3. 初始化物理数据源map

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

4. 定义分库分表规则配置

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

5. 数据源更换为ShardingDataSource后，所有的分库分表逻辑都交由ShardingSphere处理了，对用户完全透明

# 分库方案

1. 分成4个库，两个mysql实例承载，这种规模是无法承接两次大考的。
2. 考试数据有个特点，最近一个月的数据访问会很频繁（热数据），一个月前的数据很少访问（冷数据）
3. 因此，以自然年为维度，每年8个库，4个库用于存放热数据，4个库用于存放冷数据，有一个归档机制，每天定时对一个月前的数据进行归档
4. 定期（每年、每个学期、市场有重大变化）对业务规模进行评估，根据业务规模对分库个数进行扩容。
5. 分库扩容后，最好不要进行数据迁移，简化扩容过程



# 参考资料

[ShardingSphere](https://shardingsphere.apache.org/document/legacy/3.x/document/cn/overview/)