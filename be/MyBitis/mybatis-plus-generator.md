---
isPublish: false
---



# mybatis-plus-generator



## MyBatis Plus 代码生成器

**依赖**

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>latest</version>
</dependency>
```

**CodeGenerator 类**

```java
public class CodeGenerator {
    public static void main(String[] args) {
        String moduleName = "cms";	//生成的模块名
        // 1、创建代码生成器
        AutoGenerator mpg = new AutoGenerator();

        // 2、全局配置
        GlobalConfig gc = new GlobalConfig();
      	// 输出路径 需要绝对路径
        gc.setOutputDir("generate/src/main/java");
        gc.setAuthor("");
        gc.setOpen(false); // 生成后是否打开资源管理器
        gc.setFileOverride(false); // 重新生成时文件是否覆盖
        gc.setServiceName("%sService");	// 去掉 Service 接口的首字母 I
        gc.setIdType(IdType.AUTO); // 主键策略
        gc.setDateType(DateType.ONLY_DATE);// 定义生成的实体类中日期类型
        gc.setSwagger2(true);// 开启 Swagger2 模式
        gc.setBaseColumnList(true);

        mpg.setGlobalConfig(gc);

        // 3、数据源配置
        DataSourceConfig dsc = new DataSourceConfig();
        dsc.setUrl("jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&useSSL=false&characterEncoding=utf8");
        dsc.setDriverName("com.mysql.jdbc.Driver");
        dsc.setUsername("root");
        dsc.setPassword("root");
        dsc.setDbType(DbType.MYSQL);
        mpg.setDataSource(dsc);

        // 4、包配置
        PackageConfig pc = new PackageConfig();
        pc.setModuleName(moduleName); // 模块名
        pc.setParent("com.atguigu.gmall");
        pc.setController("controller");
        pc.setEntity("entity");
        pc.setService("service");
        pc.setMapper("mapper");
        mpg.setPackageInfo(pc);

        // 5、策略配置
        StrategyConfig strategy = new StrategyConfig();
        strategy.setInclude(moduleName + "_\\w*"); // 设置要映射的表名
      	// 数据库表映射到实体的命名策略
        strategy.setNaming(NamingStrategy.underline_to_camel);
        strategy.setTablePrefix(pc.getModuleName() + "_");//设置表前缀不生成
      	//是否生成实体时，生成字段注解
        strategy.setEntityTableFieldAnnotationEnable(true);

      	// 数据库表字段映射到实体的命名策略
        strategy.setColumnNaming(NamingStrategy.underline_to_camel);
      	// lombok 模型 @Accessors(chain = true) setter 链式操作
        strategy.setEntityLombokModel(true); 

        //strategy.setLogicDeleteFieldName("is_deleted");// 逻辑删除字段名
      	// 去掉布尔值的 is_ 前缀
        //strategy.setEntityBooleanColumnRemoveIsPrefix(true);

        // 自动填充
        //TableFill gmtCreate = new TableFill("gmt_create", FieldFill.INSERT);
        //TableFill gmtModified = new TableFill("gmt_modified", FieldFill.INSERT_UPDATE);
        //ArrayList<TableFill> tableFills = new ArrayList<>();
        //tableFills.add(gmtCreate);
        //tableFills.add(gmtModified);
        //strategy.setTableFillList(tableFills);

        //strategy.setVersionFieldName("version"); //乐观锁列

        strategy.setRestControllerStyle(true);  // restful api 风格控制器
        strategy.setControllerMappingHyphenStyle(true); // url 驼峰转连字符

        mpg.setStrategy(strategy);
        mpg.setTemplateEngine(new FreemarkerTemplateEngine());
        // 6、执行
        mpg.execute();
    }
}

```

3、将逆向工程生成的代码复制进对应的工程模块

① 先复制 Mapper

② 新建 `service.impl` 包

③ 将相应的实现类复制过来



<br>

## 参考

* https://baomidou.com/