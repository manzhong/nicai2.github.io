---
title: SparkSql语法词法解析
tags:
  - Spark
categories: Spark
abbrlink: 14104
date: 2020-05-31 21:46:49
summary_img:
encrypt:
enc_pwd:
---

# 一 词法,语法解析简介

SQL作为现代计算机行业的数据处理事实标准，一直倍受数据分析师和软件开发者所青睐，从最早应用SQL的单机DBMS（如MySQL、Oracle），到NoSQL提供SQL兼容层（如HBase的Phoenix），到最近火热的NewSQL（如Spanner、TiDB），还有几乎所有主流的计算框架（如Spark、Flink）都兼容了SQL。

SQL规范也一直在稳定发展，近年来通过语法拓展满足了离线批处理、流式处理以及时序数据处理几乎所有的需求，并且因为用户基数极大，兼容SQL让用户可以没有学习成本地把业务迁移到新的软件系统中。SQL本质上是一种数据处理的描述语言，或者说是一系列符合SQL标准的字符串，任何系统都可以解析SQL来生成对用平台的任务，因此SQL parser就是这些系统中最重要的组件之一，而且通过拓展SQL parser还可以实现自定义的SQL语义。那么怎样才可以实现自己的SQL parser呢，正是本文准备介绍的。

# 二 实现sql parser原理

SQL是一种描述语言的规范，有SQL86、SQL89、SQL92、SQL1999、SQL2003、SQL2006、SQL2008、SQL2011等标准，而MySQL、Oracle甚至Spark也有自己支持的SQL规范，所谓SQL兼容其实也不是和SQL社区定义的标准100%一样，和Python定义语法规范但还有CPython、Jython、Cython等多种虚拟机实现类似。而SQL描述语言相比其他编程语言更简单一些，一般分为DDL和DML，大部分计算系统只是支持DML中的查询语法，也就是SELECT FROM语句，因为对于计算系统来说不需要管理Schema的修改以及数据的增删改，如果是兼容SQL的存储系统就需要兼容几乎所有DDL和DML语句。

那么要解析用户编写的SQL字符串一般有以下几种方法：

1. 简单字符串处理，使用字符串查找或者正则表达式来提取SQL中的字段，对于简单的SQL可以这样实现，但SQL规范还有复杂的开闭括号以及嵌套查询，复杂SQL几乎不可能通过字符串匹配来实现。
2. 使用已有的开源SQL解释器，不同开源系统解析SQL后的应用是不一样的，一般会转化成自己实现的抽象语法树，这些数据结构包含自己定义的树节点以及表达式节点，要兼容使用这些数据结构是非常困难的，这也是为什么开源项目中支持SQL的很多但基本不可以复用同一套实现的原因。
3. 从头实现SQL解释器，对SQL做词法解析、语法解析的项目已经有很多了，**通过遍历抽象语法树来解析SQL实现对应的业务逻辑**，从头实现一个SQL解释器并没有想象中困难，而且还可以选择增加或减少支持的SQL语法范围。

**本文会基于这里会基于封装性更好并且更加易用的Antlr4来介绍,(Presto和Spark都是基于Antlr框架来实现SQL parser的，也就是他们没有重复造SQL词法解析器、SQL语法解析器的轮子)**

# 三 使用Antlr4基于spark环境解析sparkSql

Antlr4是一个Java实现的开源项目，用户需要编写g4后缀的语法文件，Antlr4可以自动生成词法解析器和语法解析器，提供给开发者的接口是已经解析好的抽象语法树以及易于访问的Listener和Visitor基类。什么意思呢，就是如果你要实现一个SQL parser，**只要提供一个SQL语法规范的g4描述文件，这个文件可以从Presto或Spark项目中获得**，那么Antlr就会生成编译过程中的抽象语法树，用户也只需要写一个Java类来选择感兴趣的节点接口，g4文件格式需要符合Antlr要求但因为是标准SQL我们不用自己重新写可以复用Presto或Spark的

spark的g4文件地址:

```http
https://github.com/apache/spark/blob/v2.3.1/sql/catalyst/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBase.g4
```



**基于现有的语法文件和开源库，用户只要只要传入SQL字符串，就可以马上得到SQL的抽象语法树了**。这里推荐使用IntelliJ IDEA，安装antlr插件后，输入SQL资源就可以可视化这棵抽象语法树，方面后续遍历抽象语法树实现自己的业务逻辑。

有了抽象语法树，以及Antlr自动生成的Visitor基类，用户只需要自己的Visitor类即可，这部分需要参考Antlr的使用文档，实际上也并不复杂。例如我们需要解析SQL获取用户查询的表明，从下面的抽象语法树结构看到需要遍历的节点是fromClause、relationPrimary、tableIdentifier、identifier和strictIdentifier，当然如果用户写的是“SELECT c1, c2 FROM t1 as t2”，那么还可以在tableAlias节点获取重命名的字段。

要获取这个表名并打印出来，实现的Visitor类代码也很直观，**只需要重写我们关注的几个节点的visit函数即可**，如果要支持获取其他字段只需多visit几个节点即可，示例代码如下。

```java
//在这个类里可以获取sql中的内容 具体参考官方文档
import org.antlr.v4.runtime.tree.ParseTree;
import org.apache.spark.sql.catalyst.parser.SqlBaseBaseVisitor;
import org.apache.spark.sql.catalyst.parser.SqlBaseParser;

public class SimpleSqlVisitor extends SqlBaseBaseVisitor {
  
     String tableName="";

    public String getTableName() {
        return tableName;
    }

    public void setTableName(String tableName) {
        this.tableName = tableName;
    }
	//获取drop的tablename
    @Override
    public Object visitDropTable(SqlBaseParser.DropTableContext ctx) {
        List<ParseTree> children = ctx.children;
        String droptableName="";
        for(int i=0 ;i<children.size();i++){
            ParseTree parseTree = children.get(i);
            String text = parseTree.getText();
            if(text.equalsIgnoreCase("exists")){
                droptableName=children.get(i+1).getText();
            }
        }
        System.out.println(droptableName);
        this.tableName=droptableName;
        return super.visitDropTable(ctx);
    }


//获取insert overwrite的tablename
    @Override
    public Object visitInsertOverwriteTable(SqlBaseParser.InsertOverwriteTableContext ctx) {
        List<ParseTree> children = ctx.children;
        String tableName="";
        for(int i=0 ;i<children.size();i++){
            ParseTree parseTree = children.get(i);
            String text = parseTree.getText();

            if(text.equalsIgnoreCase("table")){
                tableName=children.get(i+1).getText();
            }
        }
        this.tableName=tableName;

        System.out.println(tableName);
        return super.visitInsertOverwriteTable(ctx);
    }
//获取create的tablename
    @Override
    public Object visitCreateHiveTable(SqlBaseParser.CreateHiveTableContext ctx) {
        List<ParseTree> children = ctx.children;
        String tableName="";
        for(int i=0 ;i<children.size();i++){
            ParseTree parseTree = children.get(i);
            String text = parseTree.getText();
           // System.out.println(text+"2222");
            if(text.contains("CREATETABLE")){
                tableName=text.split("TABLE")[1];
            }
        }
        System.out.println(tableName);
        this.tableName=tableName;
        return super.visitCreateHiveTable(ctx);
    }
//获取insert into的tablename
    @Override
    public Object visitInsertIntoTable(SqlBaseParser.InsertIntoTableContext ctx) {
        List<ParseTree> children = ctx.children;
        String tableName="";
        for(int i=0 ;i<children.size();i++){
            ParseTree parseTree = children.get(i);
            String text = parseTree.getText();
            if(text.equalsIgnoreCase("table")){
                tableName=children.get(i+1).getText();
            }
        }
        this.tableName=tableName;
        System.out.println(tableName);
        return super.visitInsertIntoTable(ctx);
    }



    @Override
    public String visitFromClause(SqlBaseParser.FromClauseContext ctx) {
        String tableName = visitRelation(ctx.relation(0));
        System.out.println("SQL table name: " + tableName);
        return tableName;
    }

    @Override
    public String visitRelation(SqlBaseParser.RelationContext ctx) {
        if (ctx.relationPrimary() instanceof TableNameContext) {
            return visitTableName((TableNameContext)ctx.relationPrimary());
        }
        return "";
    }

    @Override
    public String visitTableName(SqlBaseParser.TableNameContext ctx) {
        return visitTableIdentifier(ctx.tableIdentifier());
    }

    @Override
    public String visitTableIdentifier(SqlBaseParser.TableIdentifierContext ctx) {
        return ctx.getChild(0).getText();
    }

}
```

然后我们需要输入一个SQL字符串进行测试，实现简单的Java main函数，初始化我们定制的visitor实例，然后以此调用Antlr提供的lexer和parser，示例如下。

```java
public class SqlExample {

    public static void main(String[] argv) {
        String sqlText = "SELECT C1, C2 FROM T1";
				String sqlF = sqlText.toUpperCase();
        SimpleSqlVisitor visitor = new SimpleSqlVisitor();
        SqlBaseLexer lexer = new SqlBaseLexer(CharStreams.fromString(sqlText));
        CommonTokenStream tokenStream = new CommonTokenStream(lexer);
        SqlBaseParser parser = new SqlBaseParser(tokenStream);

        parser.singleStatement().accept(visitor);
    }

}
```

如果是不需要提取SQL脚本内容，只是想实现一个SQL语法校验功能，那么直接运行visitor检查是否抛lexer或者parser异常即可，或者可以实现一个error listener，直接截取语法异常相关的信息。

```java
public class SimpleErrorListener extends BaseErrorListener {

    @Override
    public void syntaxError(Recognizer<?, ?> recognizer, Object offendingSymbol, int line,
            int charPositionInLine, String msg, RecognitionException e) {

        // super.syntaxError(recognizer, offendingSymbol, line, charPositionInLine, msg, e);
        System.out.println("Get SQL syntax error");
    }

}
```

最后需要在parser处注册使用，如果传入语法错误的SQL语句，通过回调找到我们自定义的error listener错误处理逻辑。

```java
public class SqlExample {

    public static void main(String[] argv) {
        String sqlText = "SELECT2 C1, C2 FROM T1";

        SimpleSqlVisitor visitor = new SimpleSqlVisitor();
        SqlBaseLexer lexer = new SqlBaseLexer(CharStreams.fromString(sqlText));
        CommonTokenStream tokenStream = new CommonTokenStream(lexer);
        SqlBaseParser parser = new SqlBaseParser(tokenStream);
        SimpleErrorListener errorListener = new SimpleErrorListener();
        parser.addErrorListener(errorListener);

        parser.singleStatement().accept(visitor);
    }

}
```

**在spark的项目中不需要提供g4描述文件;**

**ps: 当在spark的项目中,对于sql只想解析sql,看有无词法语法错误,可以这样写:**

```scala
 val plan =sparkSession.sessionState.sqlParser.parsePlan(sql)
 logger.info(plan)
```



# 四 Antlr4 的使用



