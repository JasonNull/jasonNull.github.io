---
title: DTS技术调研：分库分表
date: 2018-07-02 08:19:54
tags:
- 框架
- sharding-jdbc
categories:
- 框架
toc: true
---
## 介绍
定位为轻量级Java框架，在Java的JDBC层提供的额外服务。 它使用客户端直连数据库，以jar包形式提供服务，无需额外部署和依赖，可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架。
具有以下特性：
1. 适用于任何基于Java的ORM框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template或直接使用JDBC。
2. 基于任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid, HikariCP等。
3. 支持任意实现JDBC规范的数据库。目前支持MySQL，Oracle，SQLServer和PostgreSQL。

## 问题
1. 为什么不适用ocean base，tidb：
    - ob无法使用，虽然架构看起来很不错，但是奈何不开源啊，个人也无法购买。
    - tidb开源不友好型，单机实测性能过差，问题较多
2. 为什么选择sharding-jdbc
    - sharding-jdbc国产的，爱国一下
    - 没有找到其他合适的框架

<!-- more -->
## 作用
在DTS中的作用：
1. 分库分表，减少单数据库实例压力
2. 提升sql操作性能（读取速度，插入速度）
3. 主备，增加数据库的可用性（在没有购买阿里云drds的情况下，非常有用，不会带来额外的成本）

当然也有坏处：
1. 数据库结构设计上必须要支持分库分表（分库占用一个字段，分表占用一个字段）
2. 分布式事务的支持比较粗糙：分布式事务性能太慢，所以支持的叫做**最大努力送达型**事务
3. 批量插入不支持：数据会落在各个库中，分布式事务支持不是很好，所以不支持批量插入
4. etc，待以后发现
5. 发现一个，读写分离，没有数据同步功能，需要自己去做数据同步（mysql binlog）

## 使用示例
### 配置文件
```yaml
# mybatis配置不做解释
mybatis:
  mapper-locations: classpath*:mapper/*.xml
  type-aliases-package: io.github.jasonnull.shardingjdbcsample.model
  configuration:
      map-underscore-to-camel-case: true

# 以下是sharding-jdbc分库分表配置，加上主从配置例子
sharding:
  jdbc:
    # 多数据源配置
    datasource:
      # 数据源名称（逗号后不要加空格）
      names: ds0,ds0slave0,ds1,ds1slave0
        # 数据源配置，连接池，驱动，mysql连接url，用户名，密码
        ds0:
            type: org.apache.commons.dbcp.BasicDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/test0
            username: root
            password: abcd1234_
        ds0slave0:
            type: org.apache.commons.dbcp.BasicDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/test2
            username: root
            password: abcd1234_
        ds1:
            type: org.apache.commons.dbcp.BasicDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/test1
            username: root
            password: abcd1234_
        ds1slave0:
            type: org.apache.commons.dbcp.BasicDataSource
            driver-class-name: com.mysql.jdbc.Driver
            url: jdbc:mysql://localhost:3306/test3
            username: root
            password: abcd1234_
    # 分库分表配置
    config:
      sharding:
        master-slave-rules:
          # 逻辑名称
          ds0:
            # 主库的实际名称
            master-data-source-name: ds0
            # 从库的实际名称
            slave-data-source-names: ds0slave0
            # 读时候采用轮询
            loadBalanceAlgorithmType: ROUND_ROBIN
          ds1:
            master-data-source-name: ds1
            slave-data-source-names: ds1slave0
            loadBalanceAlgorithmType: ROUND_ROBIN
        default-database-strategy:
          inline:
            # 分库的确定列
            sharding-column: age
            # age对2取余，用于确定数据源
            algorithm-expression: ds$->{age % 2}
        tables:
          # 确定用户表的分表策略
          user:
            # 数据库表确定（ds0.user0,ds0.user1,ds0.user2,ds0.user3,ds1.user0,ds1.user1,ds1.user2,ds1.user3)
            actual-data-nodes: ds$->{0..1}.user$->{0..3}
            table-strategy:
              inline:
                # 分表的决定列
                sharding-column: id
                # 根据id，选择列
                algorithm-expression: user$->{id % 4}
            # 分布式主键列（需要是唯一的，内置分布式主键算法）
            key-generator-column-name: id
```
### Dao
```java
package io.github.jasonnull.shardingjdbcsample.dao;

import io.github.jasonnull.shardingjdbcsample.model.User;
import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * Description:
 *
 * @author chenbin，wb-cb306312
 * @create 2018/7/1 17:44
 */
@Component
public interface UserDao {
    List<User> findAll();

    int deleteAll();

    int insertAll(List<User> users);

    void insert(@Param("user") User users);
}

```
### Mapper
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="io.github.jasonnull.shardingjdbcsample.dao.UserDao">
    <sql id="allFields">
        id,
        name,
        age,
        sex
    </sql>

    <select id="findAll" resultType="User">
        select
        <include refid="allFields"/>
        from user
    </select>

    <delete id="deleteAll">
        delete from user
    </delete>

<!-- 无效
    <insert id="insertAll" parameterType="java.util.List">
        insert into user(
        name,
        age,
        sex
        ) values
        <foreach collection="list" item="user" separator=",">
            (
            #{user.name},
            #{user.age},
            #{user.sex}
            )
        </foreach>
    </insert> -->

    <insert id="insert" parameterType="User">
        insert into user(
            name,
            age,
            sex
        ) values
        (
            #{user.name},
            #{user.age},
            #{user.sex}
        )
    </insert>
</mapper>
```