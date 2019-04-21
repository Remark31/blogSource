---
title: Java的注解
date: 2019-03-16 19:26:50
tags: [Java]
---


# 前言

在初学各种Java框架时，对Java的注解表示十分不理解，使用各种框架时注解又是跑不掉的玩意，正好趁这个机会好好研究下注解到底是个啥

# 初识注解

注解本质上也是一种属性，类似于一个function，可以定义返回值属性是string，权限是public/private，是不是static等等，同样的可以定义注解。

``` java
public class AnnotationDemo {
    //@Test注解修饰方法A
    @Test
    public static void A(){
        System.out.println("Test.....");
    }

    //一个方法上可以拥有多个不同的注解
    @Deprecated
    @SuppressWarnings("uncheck")
    public static void B(){

    }
}
```

# 细看注解

注解的定义一般如下所示，例如@Test注解

``` java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Test {

} 
```


其中

## Target

Target规定了注解可以用到的地方，所有可用的枚举类型如下所示：

- TYPE：类、接口、枚举
- FIELD：字段（域）、枚举实例
- METHOD：方法
- PARAMETER：参数
- CONSTRUCTOR：构造函数
- LOCAL_VARIABLE：局部变量
- ANNOTATION_TYPE：注解
- PACKAGE：包
- TYPE_PARAMETER：类型参数
- TYPE_USE：类型使用

如果不规定Target，那么该注解可以用于任何地方


## Retention

约束注解的生命周期
- SOURCE：编译后就被丢弃（不会出现在.class文件中）
- CLASS：会出现在.class中，但是会被VM丢弃
- RUNTIME：运行中也会出现


## 一个简单的例子

``` java

//定义注解
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@interface IntegerVaule{
    int value() default 0;
    String name() default "";
}

//使用注解
public class QuicklyWay {

    //当只想给value赋值时,可以使用以下快捷方式
    @IntegerVaule(20)
    public int age;

    //当name也需要赋值时必须采用key=value的方式赋值
    @IntegerVaule(value = 10000,name = "MONEY")
    public int money;

}

```

可以看到注解定义中可以存放很多的变量，这部分存储的变量是可以被反射获取到的。


# Java的一些内置注解

- @Override：覆盖父类方法
``` java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

- @Deprecated：标明过时方法

``` java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE})
public @interface Deprecated {
```

- @SuppressWarnnings：有选择的关闭编译器对类、方法、成员变量、变量初始化的警告

``` java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {
    String[] value();
}
```

- @Documented 被修饰的注解会生成到javadoc中

- @Inherited 可以让注解被子类的Class对象使用getAnnotations()获取父类被@Inherited修饰的注解


``` java
instanceA.getClass().getAnnotations()
```

# 使用注解的一个例子

整体这个例子就是使用注解来创建出构建表的SQL语句

注解的定义

``` java 


// 表
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface DBTable {
    String name() default "";
}


// varchar类型
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SQLString {

    String name() default "";
    // 长度
    int value() default 0;

    Constraints constraints() default @Constraints;
}


// 整数类型
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SQLInteger {
    String name() default "";
    Constraints constraints() default @Constraints;
}

// 表的约束

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Constraints {
    // 是否主键
    boolean primaryKey() default false;

    // 是否为空
    boolean allowNull() default false;

    // 是否唯一
    boolean unique() default false;

}

```



``` java

package annotation.createsql.model;


import annotation.createsql.annotation.Constraints;
import annotation.createsql.annotation.DBTable;
import annotation.createsql.annotation.SQLInteger;
import annotation.createsql.annotation.SQLString;


// 表的实际定义
@DBTable( name = "member")
public class Member {

    @SQLString(name = "ID", value=50, constraints = @Constraints(primaryKey = true))
    private String id;

    @SQLString(name = "NAME", value = 100)
    private String name;

    @SQLInteger(name = "AGE")
    private int age;

    @SQLString(name = "DESCRIPTION" , value = 200)
    private String description;

}

```

``` java

package annotation.createsql.constructor;


import annotation.createsql.annotation.Constraints;
import annotation.createsql.annotation.DBTable;
import annotation.createsql.annotation.SQLInteger;
import annotation.createsql.annotation.SQLString;
import annotation.createsql.model.Member;

import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

/**
 * 根据注解构建表
 *
 * @author remark
 * @version 2019年03月10日23:11:08
 */
public class TableCreator {

    /**
     * 创建表
     * @param className
     * @return
     * @throws ClassNotFoundException
     */
    public static String createTableSql(String className) throws ClassNotFoundException {
        Class<?> cl = Class.forName(className);
        DBTable dbTable = cl.getAnnotation(DBTable.class);
        if (dbTable == null) {
            System.out.println("No DBTable annotations in class " + className);
            return null;
        }

        // 表名
        String tableName = dbTable.name();
        if (tableName.length() < 1) {
            tableName = cl.getName().toUpperCase();
        }

        List<String> columnsDefs = new ArrayList<>();
        // 获取成员字段

        for(Field field: cl.getDeclaredFields()) {
            String columnsName = null;

            Annotation[] annos = field.getDeclaredAnnotations();

            if (annos.length < 1) {
                continue;
            }

            if (annos[0] instanceof SQLInteger) {
                SQLInteger sqlInteger = (SQLInteger) annos[0];

                columnsName = sqlInteger.name();

                if (columnsName.length() < 1) {
                    columnsName = field.getName().toUpperCase();
                }

                columnsDefs.add(columnsName + " INT" + getConstraints(sqlInteger.constraints()));

            }

            if (annos[0] instanceof SQLString) {
                SQLString sqlString = (SQLString) annos[0];

                columnsName = sqlString.name();
                if (columnsName.length() < 1) {
                    columnsName = field.getName().toUpperCase();
                }

                columnsDefs.add(columnsName + "VARCHAR(" + sqlString.value() + ")" + getConstraints(sqlString.constraints()));
            }



        }

        StringBuilder createCommand = new StringBuilder("CREATE TABLE " + tableName + "(");

        for(String columnDef : columnsDefs){
            createCommand.append("\n" + columnDef + ",");
        }



        String tableCreate = createCommand.substring(
                0, createCommand.length() - 1) + ");";


        return tableCreate;
    }

    /**
     * 获取限制条件
     * @return
     */
    public static String getConstraints(Constraints constraints) {
        String cons = "";
        if( !constraints.allowNull()) {
            cons += " NOT NULL";
        }
        if ( constraints.primaryKey()){
            cons += " PRIMARY KEY";
        }
        if ( constraints.unique()) {
            cons += " UNIQUE";
        }

        return cons;
    }



    public static void main(String[] args) throws Exception {
        Member member = new Member();

        String className = member.getClass().getName();

        System.out.println("Table Creation SQL for " +
                className + " is :\n" + createTableSql(className));
    }
}

``` 

其实注解就是作为一个配置项被使用的，通过反射获取配置，然后进行一个通用的处理。看上去非常的方便。实际上在框架中去理解注解的功能，如果没有相关文档就非常困难了，注解的定义处只会存下一个配置，实际使用的反射的地方是很难找寻到的，但对于框架来说，这简直不能太美好。

一时注解一时爽，一直注解一直爽。




