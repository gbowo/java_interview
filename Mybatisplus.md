## Mybatis-plus


### 一、入门


#### 1. 使用MybatisPlus基本步骤

1. 引入MybatisPlus依赖，代替Mybatis依赖
2. 定义Mapper接口并继承BaseMapper(extends BaseMapper<>)


#### 2. 常见注解

**MybatisPlus的常用注解有哪些？**
- **@TableName("")**：用来指定表名
- **@TableId(value="",type=IdType.AUTO)**：用来指定表种的主键字段信息
- **@TableField(""/exist = false)**：用来指定表中的普通字段信息

**MybatisPlus是如何获取实现CRUD的数据库表信息的？**
- 默认类名驼峰转下划线作为表名
- 默认名为id的字段作为主键
- 默认变量名驼峰转下划线作为表的字段名

**idType的常见类型有哪些？**
- AUTO、ASSIGN_ID、INPUT

**使用@TableField的常见场景是？**
- 成员变量名与数据库字段名不一致
- 成员变量名以is开头，且是布尔值
- 成员变量名与数据库关键词冲突
- 成员变量不是数据库字段


#### 3. 常见配置

**MybatisPlus使用的基本流程是什么？**
1. 引入起步依赖
2. 自定义Mapper基础BaseMapper
3. 在实体类上添加注解声明 表信息
4. 在application.yml种根据需要添加配置


### 二、核心功能


#### 1. 条件构造器

**使用规则**
``` java
@Test
void testQueryWrapper(){
    // 1. 构造查询条件
    QueryWrapper<User> wrapper = new QueryWrapper<User>()
        .select("id","username","info","balance")
        .like("username","o")
        .ge("balance",1000);
    // 2. 查询
    List<User> users = userMapper.selectList(wrapper);
    users.forEach(System.out::println);
}

// 避免硬编码
@Test
void testLambdaQueryWrapper(){
    // 1. 构造查询条件
    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>()
        .select(User::getId,User::getUsername,User::getInfo,User::getBalance)
        .like(User::getUsername,"o")
        .ge(User::getBalance,1000);
    // 2. 查询
    List<User> users = userMapper.selectList(wrapper);
    users.forEach(System.out::println);
}

@Test
void testUpdateByQueryWrapper(){
    // 1. 要更新的数据
    User user = new User();
    user.setBalance(2000);
    // 2. 更新的条件
    QueryWrapper<User> wrapper = new QueryWrapper<User>().eq("username","jack");
    // 3. 执行更新
    userMapper.update(user,wrapper);
}

// 需求：更新id为1、2、4的用户的余额，扣200
@Test
void testUpdateWrapper(){
    List<Long> ids = List.of(1L,2L,4L);
    UpdateWrapper<User> wrapper = new UpdateWrapper<User>()
            .setSql("balance = balance - 200")
            .in("id",ids);
    userMapper.update(null,wrapper);
}

```


**条件构造器的用法：**
- QueryWrapper和LambdaQueryWrapper通常用来构建select、delete、update的where条件部分
- UpdateWrapper和LambdaUpdateWrapper通常只有在set语句比较特殊才使用
- 尽量使用LambdaQueryWrapper和LambdaUpdateWrapper，避免硬编码


#### 2. 自定义SQL

``` java

// @Test
// void testUpdateWrapper(){
//     List<Long> ids = List.of(1L,2L,4L);
//     UpdateWrapper<User> wrapper = new UpdateWrapper<User>()
//             .setSql("balance = balance - 200")
//             .in("id",ids);
//     userMapper.update(null,wrapper);
// }

@Test
void testCustomSqlUpdate(){
    // 1. 更新条件
    List<Long> ids = List.of(1L,2L,4L);
    int amount = 200;
    // 2. 定义条件
    QueryWrapper<User> wrapper = new QueryWrapper<User>().in("id",ids);
    // 3. 调用自定义SQL方法
    userMapper.updateBalanceByIds(wrapper,amount);
}

// 在UserMapper.java中

public interface UserMapper extends BaseMapper<User>{
    List<User> queryUserByIds(@Param("ids") List<Long> ids);

    void updateBalanceByIds(@Param(Constrant.WRAPPER) QueryWrapper<User> wrapper,int amount);

}
```
**在UserMapper.xml中更新sql语句**
``` xml
<update id="updateBalanceByIds">
    UPDATE tb_user SET balance = balance - #{amount} ${ew.customSqlSegment}
</update>

```
**我们可以利用MP的Wrapper来构建复杂的Where条件，然后自己定义SQL语句中剩下的部分**
1. 基于Wrapper构建where条件
2. 在mapper方法参数中用Param注解声明wrapper变量名称，必须是ew
3. 自定义SQL，并使用Wrapper条件


#### 3. Service接口

``` java
public interface IUserService extends IService<User>{}
```

``` java
public class UserServiceImpl extends ServiceImpl<UserMapper,User> implements IUserService{
    
}
```

**MP的Service接口使用流程是怎样的？**
- 自定义Service接口继承IService接口
- 自定义Service实现类，实现自定义接口并继承ServiceImpl类


**IService批量新增**

- 普通for循环逐条插入速度极差，不推荐
- 批处理,MP的批量新增，基于预编译的批处理，性能不错
- 配置jdbc参数，开rewriteBatchedStatements，性能最好


``` java
    @Test
    void testSaveOneByOne(){
        long b = System.currentTimeMillis();
        for(int i = 2;i <= 100000;i++){
            userService.save(buildUser(i));
        }
        long e = System.currentTimeMillis();
        System.out.println("耗时: " + (e - b));
    }

    @Test
    void testSaveBatch(){
        // 每次批量输入1000条件，插入100此即10万条数据

        // 1。 准备一个容量为1000的集合
        List<User> list = new ArrayList<>(1000);
        long b = System.currentTimeMillis();
        for (int i = 1;i <= 100000; i++){
            // 2. 添加一个user
            list.add(buildUser(i));
            // 3. 每1000条批量插入1次
            if(i % 1000 == 0){
                userService.saveBatch(list);
                // 清空集合，准备下一批数据
                list.clear();
            }
        }
        long e = System.currentTimeMillis();
        System.out.println("耗时："+(e - b));
    }
```


``` yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/mp?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password:
```


### 三、Mybatis-Plus扩展功能

#### 1. Db静态工具
- 避免循环依赖
- 用于两个及以上service相互调用的情况


#### 2. 逻辑删除


**逻辑删除**：基于代码逻辑模拟删除效果，但并不会真正删除数据
- 在表中添加一个字段标记数据书否被删除
- 当删除数据时把标记置为1
- 查询时只查询标记为0的数据
- 总结：MP提供了逻辑删除功能，无需改变方法调用的方式，而是在底层帮我们自动修改CRUD语句，我们要做的就是在application.yaml文件中配置逻辑删除的字段名称和值即可


``` yaml
mybatis-plus:
  type-aliases-package: com.itheima.mp.domain.po
  global-config:
    db-config:
      id-type: auto
      logic-delete-field: deleted # 全局逻辑删除的实体字段名，字段类型可以是boolean、integer
      logic-delete-value: 1 # 逻辑已删除值（默认为1）
      logic-not-delete-value: 0 # 逻辑未删除值（默认为0）
```

**逻辑删除的问题**：
- 会导致数据库表垃圾数据越来越多，影响查询效率
- SQL中全都需要对逻辑删除字段做判断，影响查询效率
- 不太推荐，若数据不能删除可以采用把数据迁移到其他表的方法


#### 3. 枚举转换器


**如何实现PO类中的枚举类型变量与数据库字段的转换？**
- 给枚举中的与数据库对应value值添加@EnumValue注解
- 在配置文件中配置统一的枚举处理器；实现类型转换


``` java
    @EnumValue
    private final int value;
    @JsonValue
    private final String desc;
```

``` yaml
  configuration:
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
```


### 四、插件功能


#### 分页插件


**步骤**
- 项目中新建一个配置类.config.MybatisConfig
- 编写一个分页查询的测试
- 既可以用分页参数Page也可以支持排序参数

``` java
package com.itheima.mp.config;

import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        // 初始化核心插件
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```


``` java
@Test
void testPageQuery() {
    // 1.分页查询，new Page()的两个参数分别是：页码、每页大小
    Page<User> p = userService.page(new Page<>(2, 2));
    // 2.总条数
    System.out.println("total = " + p.getTotal());
    // 3.总页数
    System.out.println("pages = " + p.getPages());
    // 4.数据
    List<User> records = p.getRecords();
    records.forEach(System.out::println);
}
```

``` java
int pageNo = 1, pageSize = 5;
// 分页参数
Page<User> page = Page.of(pageNo, pageSize);
// 排序参数, 通过OrderItem来指定
page.addOrder(new OrderItem("balance", false));

userService.page(page);
```