---
title: SparkSql-2.4.3源码解析
tags:
  - Spark
categories: Spark
encrypt: 
enc_pwd: 
abbrlink: 29326
date: 2020-06-07 18:53:01
summary_img:
---

# 一 架构概览

sparkSql 使用antlr4 解析sql ,所以用户可以基于spark引擎使用sql语句对数据进行分析，而不用去编写程序代码.

 spark sql的运行流程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190124150236498.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW9qaWFvNTIxNzY1MTQ2NTE0,size_16,color_FFFFFF,t_70)

大概有6步：

1. sql 语句经过 SqlParser 解析成 Unresolved Logical Plan;
2. analyzer 结合 catalog 进行绑定,生成 Logical Plan;
3. optimizer 对 Logical Plan 优化,生成 Optimized LogicalPlan;
4. SparkPlan 将 Optimized LogicalPlan 转换成 Physical Plan;
5. prepareForExecution()将 Physical Plan 转换成 executed Physical Plan;
6. execute()执行可执行物理计划，得到RDD;

上述流程在spark中对应的源码部分：

```scala
class QueryExecution(val sparkSession: SparkSession, val logical: LogicalPlan) {

  // TODO: Move the planner an optimizer into here from SessionState.
  protected def planner = sparkSession.sessionState.planner

  def assertAnalyzed(): Unit = analyzed

  def assertSupported(): Unit = {
    if (sparkSession.sessionState.conf.isUnsupportedOperationCheckEnabled) {
      UnsupportedOperationChecker.checkForBatch(analyzed)
    }
  }

  lazy val analyzed: LogicalPlan = {
    SparkSession.setActiveSession(sparkSession)
    sparkSession.sessionState.analyzer.executeAndCheck(logical)
  }

  lazy val withCachedData: LogicalPlan = {
    assertAnalyzed()
    assertSupported()
    sparkSession.sharedState.cacheManager.useCachedData(analyzed)
  }

  lazy val optimizedPlan: LogicalPlan = {
    sparkSession.sessionState.optimizer.execute(withCachedData)
  }

  lazy val sparkPlan: SparkPlan = {
    SparkSession.setActiveSession(sparkSession)
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
    //       but we will implement to choose the best plan.
    planner.plan(ReturnAnswer(optimizedPlan)).next()
  }

  // executedPlan should not be used to initialize any SparkPlan. It should be
  // only used for execution.
  lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)

  /** Internal version of the RDD. Avoids copies and has no schema */
  lazy val toRdd: RDD[InternalRow] = executedPlan.execute()
```

​     逻辑交代的非常清楚，从最后一行的   lazy val toRdd: RDD[InternalRow] = executedPlan.execute() 往前推便可清晰看到整个流程。细心的同学也可以看到 所有步骤都是lazy的，只有调用了execute才会触发执行，这也是spark的重要设计思想。

# 二 源码分析

当我们执行: 以SparkSession 的 sql(sqlText: String): DataFrame 为例

```scala
sparkSession.sql("select * .....")
```

## 2.1 sql的parser

看一下sql函数:

```scala
  /**
   * Executes a SQL query using Spark, returning the result as a `DataFrame`.
   * The dialect that is used for SQL parsing can be configured with 'spark.sql.dialect'.
   *
   * @since 2.0.0
   */
  def sql(sqlText: String): DataFrame = {
    Dataset.ofRows(self, sessionState.sqlParser.parsePlan(sqlText))
  }
```

```scala
/**
 * Interface for a parser.
 */
trait ParserInterface {
  /**
   * Parse a string to a [[LogicalPlan]].
   */
  //将sql转为逻辑计划
  def parsePlan(sqlText: String): LogicalPlan
  .....
```

而在抽象类AbstractSqlParser中重写了这个方法:

```scala
abstract class AbstractSqlParser extends ParserInterface with Logging {
	.
  .
  .
  /** Creates LogicalPlan for a given SQL string. */
  //这里调用了主函数parse 把sql变为LogicalPlan(逻辑计划) 
  override def parsePlan(sqlText: String): LogicalPlan = parse(sqlText) { parser =>
    //从 singleStatement 结点开始，遍历语法树，将结点转换为逻辑计划
    astBuilder.visitSingleStatement(parser.singleStatement()) match {
      case plan: LogicalPlan => plan
      case _ =>
        val position = Origin(None, None)
        throw new ParseException(Option(sqlText), "Unsupported SQL statement", position, position)
    }
  }

  /** Get the builder (visitor) which converts a ParseTree into an AST. */
  protected def astBuilder: AstBuilder
//变为逻辑计划的最终实现方法
  //使用 ANTLR 4 实现了 SQL 语句的词法分析和语法分析，获得了抽象语法树
  protected def parse[T](command: String)(toResult: SqlBaseParser => T): T = {
    logDebug(s"Parsing command: $command")
//词法解析器SqlBaseLexer(将SQL语句解析成一个个短语)
    val lexer = new SqlBaseLexer(new UpperCaseCharStream(CharStreams.fromString(command)))
    lexer.removeErrorListeners()
    lexer.addErrorListener(ParseErrorListener)
    lexer.legacy_setops_precedence_enbled = SQLConf.get.setOpsPrecedenceEnforced
    //语法解析器SqlBaseParser
    // Antlr 生成的 SqlBaseParser 进行语法分析，得到 LogicalPlan
    val tokenStream = new CommonTokenStream(lexer)
    val parser = new SqlBaseParser(tokenStream)
    parser.addParseListener(PostProcessor)
    parser.removeErrorListeners()
    parser.addErrorListener(ParseErrorListener)
    parser.legacy_setops_precedence_enbled = SQLConf.get.setOpsPrecedenceEnforced

    try {
      try {
        // first, try parsing with potentially faster SLL mode
        parser.getInterpreter.setPredictionMode(PredictionMode.SLL)
        toResult(parser)
      }
      catch {
        case e: ParseCancellationException =>
          // if we fail, parse with LL mode
          tokenStream.seek(0) // rewind input stream
          parser.reset()

          // Try Again.
          parser.getInterpreter.setPredictionMode(PredictionMode.LL)
          toResult(parser)
      }
    }
    catch {
      case e: ParseException if e.command.isDefined =>
        throw e
      case e: ParseException =>
        throw e.withCommand(command)
      case e: AnalysisException =>
        val position = Origin(e.line, e.startPosition)
        throw new ParseException(Option(command), e.message, position, position)
    }
  }
}
.
.
.
.

```

在AbstractSqlParser的实现类SparkSqlParser中:

```scala
/**
 * Concrete parser for Spark SQL statements.
 */
class SparkSqlParser(conf: SQLConf) extends AbstractSqlParser {
  val astBuilder = new SparkSqlAstBuilder(conf)

  private val substitutor = new VariableSubstitution(conf)

  protected override def parse[T](command: String)(toResult: SqlBaseParser => T): T = {
    //调用父类的parse方法
    super.parse(substitutor.substitute(command))(toResult)
  }
}
```

**最终 parsePlan 函数将 sql语句变成了 LogicalPlan **

仔细阅读 parse函数，可以发现其中主要的工作主力是SqlBaseLexer 和 SparkSqlAstBuilder,它们都是antlr4相关的代码

**要了解antlr4 可以参考我的另一篇博客:**

```http
http
```

**接下来我们看一下SparkSqlAstBuilder:**

首先我门看一下SparkSqlAstBuilder 的继承体系:

```scala
class SparkSqlAstBuilder(conf: SQLConf) extends AstBuilder(conf)

class AstBuilder(conf: SQLConf) extends SqlBaseBaseVisitor[AnyRef] with Logging 
```

参考我的博客sparksql的语法词法解析可以对SqlBaseBaseVisitor有个大致了解:

```http
http
```

**实际就是AstBuilder 封装了大量的visit开头的方法,使用这些方法对sql语句进行分段解析如(join,where, from,select等)(个人理解,可能有失偏颇)**

为了使读者,不看上面的博客对antlr4也有个大概了解,一下我简单介绍一下:

使用antlr4 ,需要的因素:

**1.语法文件(org.apache.spark.sql.catalyst.parser.SqlBase.g4，放置在子工程catalyst中)**

**2.监视器类或者访问者类**
在spark-sql的体系中,主要是使用访问者类(SparkSqlAstBuilder),但是也使用了监听器类辅助(PostProcessor)来处理格式转换

### 第一步: 从 astBuilder.visitSingleStatement结点开始，遍历语法树，将结点转换为逻辑计划

```scala
  override def visitSingleStatement(ctx: SingleStatementContext): LogicalPlan = withOrigin(ctx) {
    visit(ctx.statement).asInstanceOf[LogicalPlan]
  }
```

SqlBaseBaseVisitor 的父类就是AbstractParseTreeVisitor

```java
public abstract class AbstractParseTreeVisitor<T> implements ParseTreeVisitor<T> {
//
/**
	 * {@inheritDoc}
	 *
	 * <p>The default implementation calls {@link ParseTree#accept} on the
	 * specified tree.</p>
	 */
	@Override
	public T visit(ParseTree tree) {
		return tree.accept(this);
	}
```

```java
@Override
public <T> T accept(ParseTreeVisitor<? extends T> visitor) {
	if ( visitor instanceof SqlBaseVisitor ) return ((SqlBaseVisitor<? extends T>)visitor).visitQuerySpecification(this);
	else return visitor.visitChildren(this);
}
```

### 第二步:AstBuilder.scala 实现了SqlBaseBaseVisitor用于解析逻辑的实现

```scala
  override def visitQuerySpecification(
      ctx: QuerySpecificationContext): LogicalPlan = withOrigin(ctx) {
    val from = OneRowRelation().optional(ctx.fromClause) {
      visitFromClause(ctx.fromClause)
    }
    withQuerySpecification(ctx, from)
  }
```

FROM 语句解析

```scala
  override def visitFromClause(ctx: FromClauseContext): LogicalPlan = withOrigin(ctx) {
    val from = ctx.relation.asScala.foldLeft(null: LogicalPlan) { (left, relation) =>
      val right = plan(relation.relationPrimary)
      val join = right.optionalMap(left)(Join(_, _, Inner, None))
      withJoinRelations(join, relation)
    }
    if (ctx.pivotClause() != null) {
      if (!ctx.lateralView.isEmpty) {
        throw new ParseException("LATERAL cannot be used together with PIVOT in FROM clause", ctx)
      }
      withPivot(ctx.pivotClause, from)
    } else {
      ctx.lateralView.asScala.foldLeft(from)(withGenerate)
    }
  }
```

WHERE 等语句解析,看这很复杂,但我们只需要关注自己的sql:

```scala
/**
   * Add a query specification to a logical plan. The query specification is the core of the logical
   * plan, this is where sourcing (FROM clause), transforming (SELECT TRANSFORM/MAP/REDUCE),
   * projection (SELECT), aggregation (GROUP BY ... HAVING ...) and filtering (WHERE) takes place.
   *
   * Note that query hints are ignored (both by the parser and the builder).
   */
    private def withQuerySpecification(
      ctx: QuerySpecificationContext,
      relation: LogicalPlan): LogicalPlan = withOrigin(ctx) {
    import ctx._

    // WHERE
    def filter(ctx: BooleanExpressionContext, plan: LogicalPlan): LogicalPlan = {
      Filter(expression(ctx), plan)
    }

    def withHaving(ctx: BooleanExpressionContext, plan: LogicalPlan): LogicalPlan = {
      // Note that we add a cast to non-predicate expressions. If the expression itself is
      // already boolean, the optimizer will get rid of the unnecessary cast.
      val predicate = expression(ctx) match {
        case p: Predicate => p
        case e => Cast(e, BooleanType)
      }
      Filter(predicate, plan)
    }


    // Expressions. 也就是要查询的内容
    val expressions = Option(namedExpressionSeq).toSeq
      .flatMap(_.namedExpression.asScala)
      .map(typedVisit[Expression])

    // Create either a transform or a regular query.
    val specType = Option(kind).map(_.getType).getOrElse(SqlBaseParser.SELECT)
    specType match {
      case SqlBaseParser.MAP | SqlBaseParser.REDUCE | SqlBaseParser.TRANSFORM =>
        // Transform

        // Add where.
        val withFilter = relation.optionalMap(where)(filter)

        // Create the attributes.
        val (attributes, schemaLess) = if (colTypeList != null) {
          // Typed return columns.
          (createSchema(colTypeList).toAttributes, false)
        } else if (identifierSeq != null) {
          // Untyped return columns.
          val attrs = visitIdentifierSeq(identifierSeq).map { name =>
            AttributeReference(name, StringType, nullable = true)()
          }
          (attrs, false)
        } else {
          (Seq(AttributeReference("key", StringType)(),
            AttributeReference("value", StringType)()), true)
        }

        // Create the transform.
        ScriptTransformation(
          expressions,
          string(script),
          attributes,
          withFilter,
          withScriptIOSchema(
            ctx, inRowFormat, recordWriter, outRowFormat, recordReader, schemaLess))
      //select语句
      case SqlBaseParser.SELECT =>
        // Regular select

        // Add lateral views.
        val withLateralView = ctx.lateralView.asScala.foldLeft(relation)(withGenerate)

        // Add where.
        val withFilter = withLateralView.optionalMap(where)(filter)

        // Add aggregation or a project.
        val namedExpressions = expressions.map {
          case e: NamedExpression => e
          case e: Expression => UnresolvedAlias(e)
        }
      
        //最终返回的是 Project(namedExpressions, withFilter)，他继承了LogicalPlan
        def createProject() = if (namedExpressions.nonEmpty) {
          //sql语句返回的结果
          Project(namedExpressions, withFilter)
        } else {
          withFilter
        }

        val withProject = if (aggregation == null && having != null) {
          if (conf.getConf(SQLConf.LEGACY_HAVING_WITHOUT_GROUP_BY_AS_WHERE)) {
            // If the legacy conf is set, treat HAVING without GROUP BY as WHERE.
            withHaving(having, createProject())
          } else {
            // According to SQL standard, HAVING without GROUP BY means global aggregate.
            withHaving(having, Aggregate(Nil, namedExpressions, withFilter))
          }
        } else if (aggregation != null) {
          val aggregate = withAggregation(aggregation, namedExpressions, withFilter)
          aggregate.optionalMap(having)(withHaving)
        } else {
          // When hitting this branch, `having` must be null.
          //
          createProject()
        }

        // Distinct
        val withDistinct = if (setQuantifier() != null && setQuantifier().DISTINCT() != null) {
          Distinct(withProject)
        } else {
          withProject
        }

        // Window
        val withWindow = withDistinct.optionalMap(windows)(withWindows)

        // Hint
        hints.asScala.foldRight(withWindow)(withHints)
    }
  }
```

### 第三步:最终返回的是 Project(namedExpressions, withFilter)，继承了LogicalPlan

```scala
case class Project(projectList: Seq[NamedExpression], child: LogicalPlan) extends UnaryNode {
  override def output: Seq[Attribute] = projectList.map(_.toAttribute)
  override def maxRows: Option[Long] = child.maxRows

  override lazy val resolved: Boolean = {
    val hasSpecialExpressions = projectList.exists ( _.collect {
        case agg: AggregateExpression => agg
        case generator: Generator => generator
        case window: WindowExpression => window
      }.nonEmpty
    )

    !expressions.exists(!_.resolved) && childrenResolved && !hasSpecialExpressions
  }

  override def validConstraints: Set[Expression] =
    child.constraints.union(getAliasedConstraints(projectList))
}
```

SparkSQL是通过AntlrV4这个开源解析框架解析的.在使用的时候做了几层抽象和封装：

1.构造这模式抽象AstBuilder，将AntlrV4的SQL语法实现细节封装
2.封装SparksqlParser ，并使用构造这模式，封装SparkSqlAstbuilder继承AstBuilder，那么AntlrV4的特性就都能被SparkSqlParser使用。
3.为了兼容用户自定义的解析器，解析器定定一个为ParseInferface接口类型。用户可以通过SparkSession注入自己的解析器，Spark底层会将用户的解析器和SPark的解析器做合并和统一，并保证完整性。

**然后我回到最初的SQL:**

```SCALA
 def sql(sqlText: String): DataFrame = {
    Dataset.ofRows(self, sessionState.sqlParser.parsePlan(sqlText))
  }
```

分析完了parsePlan,接下来我们看看ofRows方法:

```scala
  def ofRows(sparkSession: SparkSession, logicalPlan: LogicalPlan): DataFrame = {
     //QueryExecution创建
    val qe = sparkSession.sessionState.executePlan(logicalPlan)
    qe.assertAnalyzed()
    new Dataset[Row](sparkSession, qe, RowEncoder(qe.analyzed.schema))
  }
```

**executePlan(logicalPlan)方法创建QueryExecution**

```scala
def executePlan(plan: LogicalPlan): QueryExecution = createQueryExecution(plan)
```

**createQueryExecution(plan)创建QueryExecution**

```scala
/**
 *  QueryExecution是Spark用来执行关系型查询的主要工作流。
 *  它是被设计用来为开发人员提供更方便的对查询执行中间阶段的访问。
 *  QueryExecution中最重要的是成员变量，它的成员变量几乎都是lazy variable，
 *  它的方法大部分是为了提供给这些lazy variable去调用的
 */
class QueryExecution(val sparkSession: SparkSession, val logical: LogicalPlan) {

  // TODO: Move the planner an optimizer into here from SessionState.
  protected def planner = sparkSession.sessionState.planner

  def assertAnalyzed(): Unit = analyzed

  def assertSupported(): Unit = {
    if (sparkSession.sessionState.conf.isUnsupportedOperationCheckEnabled) {
      UnsupportedOperationChecker.checkForBatch(analyzed)
    }
  }

  //调用analyzer解析器,对Unresolved LogicalPlan进行元数据绑定生成的Resolved LogicalPlan
  lazy val analyzed: LogicalPlan = {
    SparkSession.setActiveSession(sparkSession)
    sparkSession.sessionState.analyzer.executeAndCheck(logical)
  }

  // 如果缓存中有查询结果，则直接替换为缓存的结果
  lazy val withCachedData: LogicalPlan = {
    assertAnalyzed()
    assertSupported()
    sparkSession.sharedState.cacheManager.useCachedData(analyzed)
  }

  //调用optimizer优化器
  lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)

  //已经进行过优化的逻辑执行计划进行转换而得到的物理执行计划SparkPlan
  lazy val sparkPlan: SparkPlan = {
    SparkSession.setActiveSession(sparkSession)
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
    //       but we will implement to choose the best plan.
   // 在一系列plan中选取一个
    planner.plan(ReturnAnswer(optimizedPlan)).next()
  }

  // executedPlan should not be used to initialize any SparkPlan. It should be
  // only used for execution.
  lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)

  /** Internal version of the RDD. Avoids copies and has no schema */
  //QueryExecution最后生成的RDD,触发了action操作后才会执行
  lazy val toRdd: RDD[InternalRow] = executedPlan.execute()

......

}
```

## 2.2 analyzer解析器

parser生成逻辑执行计划后，使用analyzer将逻辑执行计划进行分析。

### 第一步:sparkSession.sessionState.analyzer.executeAndCheck(logical)方法

源码地址：org.apache.spark.sql.catalyst.analysis.Analyzer.scala

```scala
class Analyzer(
    //管理着临时表、view、函数及外部依赖元数据（如hive metastore），是analyzer进行绑定的桥梁
    catalog: SessionCatalog,
    conf: SQLConf,
    maxIterations: Int)
  extends RuleExecutor[LogicalPlan] with CheckAnalysis {

  def this(catalog: SessionCatalog, conf: SQLConf) = {
    this(catalog, conf, conf.optimizerMaxIterations)
  }

  def executeAndCheck(plan: LogicalPlan): LogicalPlan = AnalysisHelper.markInAnalyzer {
    // 执行 analyze逻辑
    val analyzed = execute(plan)
    try {
      checkAnalysis(analyzed)
      analyzed
    } catch {
      case e: AnalysisException =>
        val ae = new AnalysisException(e.message, e.line, e.startPosition, Option(analyzed))
        ae.setStackTrace(e.getStackTrace)
        throw ae
    }
  }

}
```

### 第二步:execute(plan)方法

```scala
  override def execute(plan: LogicalPlan): LogicalPlan = {
    AnalysisContext.reset()
    try {
      executeSameContext(plan)
    } finally {
      AnalysisContext.reset()
    }
  }
  
  private def executeSameContext(plan: LogicalPlan): LogicalPlan = super.execute(plan)
```

### 第三步:super.execute(plan)方法

```scala
/**
 * 优化规则执行器
 * 按批次，按顺序对LogicalPlan执行rule，会迭代多次
 */
abstract class RuleExecutor[TreeType <: TreeNode[_]] extends Logging {

  /**
   * An execution strategy for rules that indicates the maximum number of executions. If the
   * execution reaches fix point (i.e. converge) before maxIterations, it will stop.
   */
  /** 控制运行次数的策略，如果在达到最大迭代次数前到达稳定点就停止运行 */
  abstract class Strategy { def maxIterations: Int }

  /** A strategy that only runs once. */
  /** 只运行一次的策略 */
  case object Once extends Strategy { val maxIterations = 1 }

  /** A strategy that runs until fix point or maxIterations times, whichever comes first. */
   /** 运行到稳定点或者最大迭代次数的策略，2选1 */
  case class FixedPoint(maxIterations: Int) extends Strategy

  /** A batch of rules. */
  //  一个批次的rules
  protected case class Batch(name: String, strategy: Strategy, rules: Rule[TreeType]*)

  /** Defines a sequence of rule batches, to be overridden by the implementation. */
  //  所有的rule，按批次存放
  protected def batches: Seq[Batch]

  /**
   * Defines a check function that checks for structural integrity of the plan after the execution
   * of each rule. For example, we can check whether a plan is still resolved after each rule in
   * `Optimizer`, so we can catch rules that return invalid plans. The check function returns
   * `false` if the given plan doesn't pass the structural integrity check.
   */
  protected def isPlanIntegral(plan: TreeType): Boolean = true

  /**
   * Executes the batches of rules defined by the subclass. The batches are executed serially
   * using the defined execution strategy. Within each batch, rules are also executed serially.
   */
  // 执行rule，关键代码很简单，就是按批次，按顺序对plan执行rule，会迭代多次
  def execute(plan: TreeType): TreeType = {
    // 用来对比执行规则前后，初始的plan有无变化
    var curPlan = plan
    val queryExecutionMetrics = RuleExecutor.queryExecutionMeter

    batches.foreach { batch =>
      val batchStartPlan = curPlan
      var iteration = 1
      var lastPlan = curPlan
      var continue = true

      // Run until fix point (or the max number of iterations as specified in the strategy.
      // 执行直到达到稳定点或者最大迭代次数
      while (continue) {
        curPlan = batch.rules.foldLeft(curPlan) {
          case (plan, rule) =>
            val startTime = System.nanoTime()
            // 执行rule，得到新的plan
            val result = rule(plan)
            val runTime = System.nanoTime() - startTime
            // 判断rule是否起了作用
            if (!result.fastEquals(plan)) {
              queryExecutionMetrics.incNumEffectiveExecution(rule.ruleName)
              queryExecutionMetrics.incTimeEffectiveExecutionBy(rule.ruleName, runTime)
              logTrace(
                s"""
                  |=== Applying Rule ${rule.ruleName} ===
                  |${sideBySide(plan.treeString, result.treeString).mkString("\n")}
                """.stripMargin)
            }
            queryExecutionMetrics.incExecutionTimeBy(rule.ruleName, runTime)
            queryExecutionMetrics.incNumExecution(rule.ruleName)

            // Run the structural integrity checker against the plan after each rule.
            if (!isPlanIntegral(result)) {
              val message = s"After applying rule ${rule.ruleName} in batch ${batch.name}, " +
                "the structural integrity of the plan is broken."
              throw new TreeNodeException(result, message, null)
            }

            result
        }
        iteration += 1
        // 到达最大迭代次数, 不再执行优化
        if (iteration > batch.strategy.maxIterations) {
          // Only log if this is a rule that is supposed to run more than once.
          // 只对最大迭代次数大于1的情况打log
          if (iteration != 2) {
            val message = s"Max iterations (${iteration - 1}) reached for batch ${batch.name}"
            if (Utils.isTesting) {
              throw new TreeNodeException(curPlan, message, null)
            } else {
              logWarning(message)
            }
          }
          continue = false
        }
        // plan不变了，到达稳定点，不再执行优化
        if (curPlan.fastEquals(lastPlan)) {
          logTrace(
            s"Fixed point reached for batch ${batch.name} after ${iteration - 1} iterations.")
          continue = false
        }
        lastPlan = curPlan
      }
      // 该批次rule是否起作用
      if (!batchStartPlan.fastEquals(curPlan)) {
        logDebug(
          s"""
            |=== Result of Batch ${batch.name} ===
            |${sideBySide(batchStartPlan.treeString, curPlan.treeString).mkString("\n")}
          """.stripMargin)
      } else {
        logTrace(s"Batch ${batch.name} has no effect.")
      }
    }

    curPlan
  }
}
```

### 第四步:rule(plan)

```scala
/**
 * 输入为旧的plan，输出为新的plan，仅此而已。
 * 所以真正的逻辑在各个继承实现的rule里，analyze的过程也就是执行各个rule的过程
 */
abstract class Rule[TreeType <: TreeNode[_]] extends Logging {

  /** Name for this rule, automatically inferred based on class name. */
  val ruleName: String = {
    val className = getClass.getName
    if (className endsWith "$") className.dropRight(1) else className
  }

  def apply(plan: TreeType): TreeType
}
```

### 第五步:rule的集合(转换规则)

```scala
  lazy val batches: Seq[Batch] = Seq(
    Batch("Hints", fixedPoint,
      new ResolveHints.ResolveBroadcastHints(conf),
      ResolveHints.ResolveCoalesceHints,
      ResolveHints.RemoveAllHints),
    Batch("Simple Sanity Check", Once,
      LookupFunctions),
    Batch("Substitution", fixedPoint,
      CTESubstitution,
      WindowsSubstitution,
      EliminateUnions,
      new SubstituteUnresolvedOrdinals(conf)),
    Batch("Resolution", fixedPoint,
      ResolveTableValuedFunctions ::
        //通过catalog解析表名 
        ResolveRelations ::
        //解析从子节点的操作生成的属性，一般是别名引起的，比如a.id 
        ResolveReferences ::
        ResolveCreateNamedStruct ::
        ResolveDeserializer ::
        ResolveNewInstance ::
        ResolveUpCast ::
        ResolveGroupingAnalytics ::
        ResolvePivot ::
        ResolveOrdinalInOrderByAndGroupBy ::
        ResolveAggAliasInGroupBy ::
        ResolveMissingReferences ::
        ExtractGenerator ::
        ResolveGenerate ::
        //解析函数
        ResolveFunctions ::
        ResolveAliases ::
        ResolveSubquery ::
        ResolveSubqueryColumnAliases ::
        ResolveWindowOrder ::
        ResolveWindowFrame ::
        ResolveNaturalAndUsingJoin ::
        ResolveOutputRelation ::
        ExtractWindowExpressions ::
        //解析全局的聚合函数，比如select sum(score) from table 
        GlobalAggregates ::
        ResolveAggregateFunctions ::
        TimeWindowing ::
        ResolveInlineTables(conf) ::
        //解析having子句后面的聚合过滤条件，比如having sum(score) > 400
        ResolveHigherOrderFunctions(catalog) ::
        ResolveLambdaVariables(conf) ::
        ResolveTimeZone(conf) ::
        ResolveRandomSeed ::
        TypeCoercion.typeCoercionRules(conf) ++
        extendedResolutionRules: _*),
    Batch("Post-Hoc Resolution", Once, postHocResolutionRules: _*),
    Batch("View", Once,
      AliasViewChild(conf)),
    Batch("Nondeterministic", Once,
      PullOutNondeterministic),
    Batch("UDF", Once,
      HandleNullInputsForUDF),
    Batch("FixNullability", Once,
      FixNullability),
    Batch("Subquery", Once,
      UpdateOuterReferences),
    Batch("Cleanup", fixedPoint,
      CleanupAliases))
```

### 第六步:ResolveRelations 通过catalog解析表名

```scala
  object ResolveRelations extends Rule[LogicalPlan] {

    // If the unresolved relation is running directly on files, we just return the original
    // UnresolvedRelation, the plan will get resolved later. Else we look up the table from catalog
    // and change the default database name(in AnalysisContext) if it is a view.
    // We usually look up a table from the default database if the table identifier has an empty
    // database part, for a view the default database should be the currentDb when the view was
    // created. When the case comes to resolving a nested view, the view may have different default
    // database with that the referenced view has, so we need to use
    // `AnalysisContext.defaultDatabase` to track the current default database.
    // When the relation we resolve is a view, we fetch the view.desc(which is a CatalogTable), and
    // then set the value of `CatalogTable.viewDefaultDatabase` to
    // `AnalysisContext.defaultDatabase`, we look up the relations that the view references using
    // the default database.
    // For example:
    // |- view1 (defaultDatabase = db1)
    //   |- operator
    //     |- table2 (defaultDatabase = db1)
    //     |- view2 (defaultDatabase = db2)
    //        |- view3 (defaultDatabase = db3)
    //   |- view4 (defaultDatabase = db4)
    // In this case, the view `view1` is a nested view, it directly references `table2`, `view2`
    // and `view4`, the view `view2` references `view3`. On resolving the table, we look up the
    // relations `table2`, `view2`, `view4` using the default database `db1`, and look up the
    // relation `view3` using the default database `db2`.
    //
    // Note this is compatible with the views defined by older versions of Spark(before 2.2), which
    // have empty defaultDatabase and all the relations in viewText have database part defined.
    // 在Catalog中匹配table信息
    def resolveRelation(plan: LogicalPlan): LogicalPlan = plan match {
      //不是这种情况 select * from parquet.`/path/to/query`  
      case u: UnresolvedRelation if !isRunningDirectlyOnFiles(u.tableIdentifier) =>
        // 默认数据库
        val defaultDatabase = AnalysisContext.get.defaultDatabase
        // 在Catalog中查找
        val foundRelation = lookupTableFromCatalog(u, defaultDatabase)
        resolveRelation(foundRelation)
      // The view's child should be a logical plan parsed from the `desc.viewText`, the variable
      // `viewText` should be defined, or else we throw an error on the generation of the View
      // operator.
      case view @ View(desc, _, child) if !child.resolved =>
        // Resolve all the UnresolvedRelations and Views in the child.
        val newChild = AnalysisContext.withAnalysisContext(desc.viewDefaultDatabase) {
          if (AnalysisContext.get.nestedViewDepth > conf.maxNestedViewDepth) {
            view.failAnalysis(s"The depth of view ${view.desc.identifier} exceeds the maximum " +
              s"view resolution depth (${conf.maxNestedViewDepth}). Analysis is aborted to " +
              s"avoid errors. Increase the value of ${SQLConf.MAX_NESTED_VIEW_DEPTH.key} to work " +
              "around this.")
          }
          executeSameContext(child)
        }
        view.copy(child = newChild)
      case p @ SubqueryAlias(_, view: View) =>
        val newChild = resolveRelation(view)
        p.copy(child = newChild)
      case _ => plan
    }

    // rule的入口，resolveOperatorsUp 后序遍历树，并对每个节点应用rule
    def apply(plan: LogicalPlan): LogicalPlan = plan.resolveOperatorsUp {
      case i @ InsertIntoTable(u: UnresolvedRelation, parts, child, _, _) if child.resolved =>
        EliminateSubqueryAliases(lookupTableFromCatalog(u)) match {
          case v: View =>
            u.failAnalysis(s"Inserting into a view is not allowed. View: ${v.desc.identifier}.")
          case other => i.copy(table = other)
        }
      // 匹配到UnresolvedRelation
      case u: UnresolvedRelation => resolveRelation(u)
    }

    // Look up the table with the given name from catalog. The database we used is decided by the
    // precedence:
    // 1. Use the database part of the table identifier, if it is defined;
    // 2. Use defaultDatabase, if it is defined(In this case, no temporary objects can be used,
    //    and the default database is only used to look up a view);
    // 3. Use the currentDb of the SessionCatalog.
    private def lookupTableFromCatalog(
      u: UnresolvedRelation,
      defaultDatabase: Option[String] = None): LogicalPlan = {
      val tableIdentWithDb = u.tableIdentifier.copy(
        database = u.tableIdentifier.database.orElse(defaultDatabase))
      try {
        catalog.lookupRelation(tableIdentWithDb)
      } catch {
        // 如果没有找到表，便会抛出异常了
        case e: NoSuchTableException =>
          u.failAnalysis(s"Table or view not found: ${tableIdentWithDb.unquotedString}", e)
        // If the database is defined and that database is not found, throw an AnalysisException.
        // Note that if the database is not defined, it is possible we are looking up a temp view.
        case e: NoSuchDatabaseException =>
          u.failAnalysis(s"Table or view not found: ${tableIdentWithDb.unquotedString}, the " +
            s"database ${e.db} doesn't exist.", e)
      }
    }
```

```scala
  //相当于后序遍历树
  def resolveOperatorsUp(rule: PartialFunction[LogicalPlan, LogicalPlan]): LogicalPlan = {
    //plan是否已经被处理过
    if (!analyzed) {
      AnalysisHelper.allowInvokingTransformsInAnalyzer {
        //递归调用,优先处理它的子节点
        val afterRuleOnChildren = mapChildren(_.resolveOperatorsUp(rule))
        if (self fastEquals afterRuleOnChildren) {
          CurrentOrigin.withOrigin(origin) {
            rule.applyOrElse(self, identity[LogicalPlan])
          }
        } else {
          CurrentOrigin.withOrigin(origin) {
            rule.applyOrElse(afterRuleOnChildren, identity[LogicalPlan])
          }
        }
      }
    } else {
      self
    }
  }
```

```scala
//catalog 存储了spark sql的所有数据表信息
class SessionCatalog(
    externalCatalogBuilder: () => ExternalCatalog,
    globalTempViewManagerBuilder: () => GlobalTempViewManager,
    functionRegistry: FunctionRegistry,
    conf: SQLConf,
    hadoopConf: Configuration,
    parser: ParserInterface,
    functionResourceLoader: FunctionResourceLoader) extends Logging {

......

    def lookupRelation(name: TableIdentifier): LogicalPlan = {
    synchronized {
      val db = formatDatabaseName(name.database.getOrElse(currentDb))
      val table = formatTableName(name.table)
      //若db等于globalTempViewManager.database
      if (db == globalTempViewManager.database) {
        //globalTempViewManager(HashMap[String, LogicalPlan])维护了一个全局viewName和其元数据LogicalPlan的映射
        globalTempViewManager.get(table).map { viewDef =>
          SubqueryAlias(table, db, viewDef)
        }.getOrElse(throw new NoSuchTableException(db, table))
      //若database已定义，且临时表中未有此table
      } else if (name.database.isDefined || !tempViews.contains(table)) {
        //从externalCatalog(如hive)中获取table对应的元数据信息metadata:CatalogTable，
        //此对象包含了table对应的类型（table（内部还是外部表），view）、存储格式、字段shema信息等
        val metadata = externalCatalog.getTable(db, table)
        //返回的table是View类型
        if (metadata.tableType == CatalogTableType.VIEW) {
          val viewText = metadata.viewText.getOrElse(sys.error("Invalid view without text."))
          // The relation is a view, so we wrap the relation by:
          // 1. Add a [[View]] operator over the relation to keep track of the view desc;
          // 2. Wrap the logical plan in a [[SubqueryAlias]] which tracks the name of the view.
          /**
           * 构造View对象（包括将viewText通过parser模块解析成语法树），并传入构造一个SubqueryAlias返回
           * 
           * 说明此table名对应的就是一个如hive的table表，通过metadata、数据和分区列的schema构造了CatalogRelation，
           * 并以此tableRelation构造SubqueryAlias返回。
           * 
           * 这里就可以看出从一个未绑定的UnresolvedRelation到通过catalog替换的过程。
           */
          val child = View(
            desc = metadata,
            output = metadata.schema.toAttributes,
            child = parser.parsePlan(viewText))
          SubqueryAlias(table, db, child)
        } else {
          SubqueryAlias(table, db, UnresolvedCatalogRelation(metadata))
        }
      } else {
        //明是个session级别的临时表，从tempTables获取到包含元数据信息的LogicalPlan 并构造SubqueryAlias返回
        SubqueryAlias(table, tempViews(table))
      }
    }
  }
......

}
```

**总结：**
Analyzer 分析前后的 LogicalPlan 对比

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190129155353223.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW9qaWFvNTIxNzY1MTQ2NTE0,size_16,color_FFFFFF,t_70)

分析后，每张表对应的字段集，字段类型，数据存储位置都已确定。Project 与 Filter 操作的字段类型以及在表中的位置也已确定。

有了这些信息，已经可以直接将该 LogicalPlan 转换为 Physical Plan 进行执行。

但是由于不同用户提交的 SQL 质量不同，直接执行会造成不同用户提交的语义相同的不同 SQL 执行效率差距甚远。换句话说，如果要保证较高的执行效率，用户需要做大量的 SQL 优化，使用体验大大降低。

为了尽可能保证无论用户是否熟悉 SQL 优化，提交的 SQL 质量如何， Spark SQL 都能以较高效率执行，还需在执行前进行 LogicalPlan 优化。

## 2.3 Optimizer优化器

optimizer是catalyst中关键的一个部分，提供对sql查询的一个优化。optimizer的主要职责是针对Analyzer的resolved logical plan，根据不同的batch优化策略)，来对执行计划树进行优化，优化逻辑计划节点(Logical Plan)以及表达式(Expression)，同时，此部分也是转换成物理执行计划的前置。

### 第一步:sparkSession.sessionState.optimizer.execute(withCachedData)方法

源码地址：org.apache.spark.sql.catalyst.optimizer.Optimizer.scala

```scala
/**
 * Optimizer的主要职责是将Analyzer给Resolved的Logical Plan根据不同的优化策略Batch，
 * 来对语法树进行优化，优化逻辑计划节点(Logical Plan)以及表达式(Expression)， 也是转换成物理执行计划的前置。
 * 它的工作原理和analyzer一致，也是通过其下的batch里面的Rule[LogicalPlan]来进行处理的
 */
abstract class Optimizer(sessionCatalog: SessionCatalog)
  extends RuleExecutor[LogicalPlan] {

  // Check for structural integrity of the plan in test mode. Currently we only check if a plan is
  // still resolved after the execution of each rule.
  override protected def isPlanIntegral(plan: LogicalPlan): Boolean = {
    !Utils.isTesting || plan.resolved
  }

  protected def fixedPoint = FixedPoint(SQLConf.get.optimizerMaxIterations)

  /**
   * Defines the default rule batches in the Optimizer.
   *
   * Implementations of this class should override this method, and [[nonExcludableRules]] if
   * necessary, instead of [[batches]]. The rule batches that eventually run in the Optimizer,
   * i.e., returned by [[batches]], will be (defaultBatches - (excludedRules - nonExcludableRules)).
   */
  def defaultBatches: Seq[Batch] = {
    val operatorOptimizationRuleSet =
      Seq(
        // Operator push down
        PushProjectionThroughUnion, //谓词下推
        ReorderJoin,
        EliminateOuterJoin,
        PushPredicateThroughJoin,
        PushDownPredicate,
        LimitPushDown,
        ColumnPruning, //列剪裁
        InferFiltersFromConstraints,
        // Operator combine
        CollapseRepartition,
        CollapseProject,
        CollapseWindow,
        CombineFilters, //合并filter
        CombineLimits, //合并limit
        CombineUnions,
        // Constant folding and strength reduction
        NullPropagation, //null处理，NULL容易产生数据倾斜
        ConstantPropagation,
        FoldablePropagation,
        OptimizeIn, // 关键字in的优化，替代为InSet
        ConstantFolding, //针对常量的优化，在这里会直接计算可以获得的常量
        ReorderAssociativeOperator,
        LikeSimplification, //like的简单优化
        //简化过滤条件，比如true and score > 0 直接替换成score > 
        BooleanSimplification,
        //简化filter，比如where 1=1 或者where 1=2，前者直接去掉这个过滤
        SimplifyConditionals,
        RemoveDispensableExpressions,
        SimplifyBinaryComparison,
        PruneFilters,
        EliminateSorts,
        //简化转换，比如两个比较字段的数据类型是一样的，就不需要转换
        SimplifyCasts,
        //简化大小写转换，比如Upper(Upper('a'))转为认为是Upper('a')
        SimplifyCaseConversionExpressions,
        RewriteCorrelatedScalarSubquery,
        EliminateSerialization,
        RemoveRedundantAliases,
        RemoveRedundantProject,
        SimplifyExtractValueOps,
        CombineConcats) ++
        extendedOperatorOptimizationRules

    val operatorOptimizationBatch: Seq[Batch] = {
      val rulesWithoutInferFiltersFromConstraints =
        operatorOptimizationRuleSet.filterNot(_ == InferFiltersFromConstraints)
      Batch("Operator Optimization before Inferring Filters", fixedPoint, //精度优化
        rulesWithoutInferFiltersFromConstraints: _*) ::
        Batch("Infer Filters", Once,
          InferFiltersFromConstraints) ::
        Batch("Operator Optimization after Inferring Filters", fixedPoint,
          rulesWithoutInferFiltersFromConstraints: _*) :: Nil
    }

    (Batch("Eliminate Distinct", Once, EliminateDistinct) ::
      // Technically some of the rules in Finish Analysis are not optimizer rules and belong more
      // in the analyzer, because they are needed for correctness (e.g. ComputeCurrentTime).
      // However, because we also use the analyzer to canonicalized queries (for view definition),
      // we do not eliminate subqueries or compute current time in the analyzer.
      Batch("Finish Analysis", Once,
        EliminateSubqueryAliases,
        EliminateView,
        ReplaceExpressions,
        ComputeCurrentTime,
        GetCurrentDatabase(sessionCatalog),
        RewriteDistinctAggregates,
        ReplaceDeduplicateWithAggregate) ::
        //////////////////////////////////////////////////////////////////////////////////////////
        // Optimizer rules start here
        //////////////////////////////////////////////////////////////////////////////////////////
        // - Do the first call of CombineUnions before starting the major Optimizer rules,
        //   since it can reduce the number of iteration and the other rules could add/move
        //   extra operators between two adjacent Union operators.
        // - Call CombineUnions again in Batch("Operator Optimizations"),
        //   since the other rules might make two separate Unions operators adjacent.
        Batch("Union", Once,
          CombineUnions) ::
        // run this once earlier. this might simplify the plan and reduce cost of optimizer.
        // for example, a query such as Filter(LocalRelation) would go through all the heavy
        // optimizer rules that are triggered when there is a filter
        // (e.g. InferFiltersFromConstraints). if we run this batch earlier, the query becomes just
        // LocalRelation and does not trigger many rules
        Batch("LocalRelation early", fixedPoint,
          ConvertToLocalRelation,
          PropagateEmptyRelation) ::
        Batch("Pullup Correlated Expressions", Once,
          PullupCorrelatedPredicates) ::
        Batch("Subquery", Once,
          OptimizeSubqueries) ::
        Batch("Replace Operators", fixedPoint,
          RewriteExceptAll,
          RewriteIntersectAll,
          ReplaceIntersectWithSemiJoin,
          ReplaceExceptWithFilter,
          ReplaceExceptWithAntiJoin,
          ReplaceDistinctWithAggregate) :: // aggregate替换distinct
        Batch("Aggregate", fixedPoint,
          RemoveLiteralFromGroupExpressions,
          RemoveRepetitionFromGroupExpressions) :: Nil ++
        operatorOptimizationBatch) :+
        Batch("Join Reorder", Once,
          CostBasedJoinReorder) :+
        Batch("Remove Redundant Sorts", Once,
          RemoveRedundantSorts) :+
        Batch("Decimal Optimizations", fixedPoint,
          DecimalAggregates) :+
        Batch("Object Expressions Optimization", fixedPoint,
          EliminateMapObjects,
          CombineTypedFilters) :+
        Batch("LocalRelation", fixedPoint,
          ConvertToLocalRelation,
          PropagateEmptyRelation) :+
        Batch("Extract PythonUDF From JoinCondition", Once,
          PullOutPythonUDFInJoinCondition) :+
        // The following batch should be executed after batch "Join Reorder" "LocalRelation" and
        // "Extract PythonUDF From JoinCondition".
        Batch("Check Cartesian Products", Once,
          CheckCartesianProducts) :+
        Batch("RewriteSubquery", Once,
          RewritePredicateSubquery,
          ColumnPruning,//去掉一些用不上的列
          CollapseProject,
          RemoveRedundantProject) :+
        Batch("UpdateAttributeReferences", Once,
          UpdateNullabilityInAttributeReferences)
  }
```

### 第二步:PushPredicateThroughJoin的关键代码

```scala
object PushPredicateThroughJoin extends Rule[LogicalPlan] with PredicateHelper {

......

  def apply(plan: LogicalPlan): LogicalPlan = plan transform {
    // push the where condition down into join filter
   // match 这个结构  
  case f @ Filter(filterCondition, Join(left, right, joinType, joinCondition)) =>
      val (leftFilterConditions, rightFilterConditions, commonFilterCondition) =
        split(splitConjunctivePredicates(filterCondition), left, right)
      joinType match {
        // 是 inner join
        case _: InnerLike =>
          // push down the single side `where` condition into respective sides
          // left下推为Filter
          val newLeft = leftFilterConditions.
            reduceLeftOption(And).map(Filter(_, left)).getOrElse(left)
          // right下推为Filter
          val newRight = rightFilterConditions.
            reduceLeftOption(And).map(Filter(_, right)).getOrElse(right)
          val (newJoinConditions, others) =
            commonFilterCondition.partition(canEvaluateWithinJoin)
          val newJoinCond = (newJoinConditions ++ joinCondition).reduceLeftOption(And)
          // 最终的优化结果
          val join = Join(newLeft, newRight, joinType, newJoinCond)
          if (others.nonEmpty) {
            Filter(others.reduceLeft(And), join)
          } else {
            join
          }
        case RightOuter =>
          // push down the right side only `where` condition
          val newLeft = left
          val newRight = rightFilterConditions.
            reduceLeftOption(And).map(Filter(_, right)).getOrElse(right)
          val newJoinCond = joinCondition
          val newJoin = Join(newLeft, newRight, RightOuter, newJoinCond)

          (leftFilterConditions ++ commonFilterCondition).
            reduceLeftOption(And).map(Filter(_, newJoin)).getOrElse(newJoin)
        case LeftOuter | LeftExistence(_) =>
          // push down the left side only `where` condition
          val newLeft = leftFilterConditions.
            reduceLeftOption(And).map(Filter(_, left)).getOrElse(left)
          val newRight = right
          val newJoinCond = joinCondition
          val newJoin = Join(newLeft, newRight, joinType, newJoinCond)

          (rightFilterConditions ++ commonFilterCondition).
            reduceLeftOption(And).map(Filter(_, newJoin)).getOrElse(newJoin)
        case FullOuter => f // DO Nothing for Full Outer Join
        case NaturalJoin(_) => sys.error("Untransformed NaturalJoin node")
        case UsingJoin(_, _) => sys.error("Untransformed Using join node")
      }

  ......

}
```

总结 :

Spark SQL 目前的优化主要是基于规则的优化，即 RBO （Rule-based optimization）

- 每个优化以 Rule 的形式存在，每条 Rule 都是对 Analyzed Plan 的等价转换
- RBO 设计良好，易于扩展，新的规则可以非常方便地嵌入进 Optimizer
- RBO 目前已经足够好，但仍然需要更多规则来 cover 更多的场景
- 优化思路主要是减少参与计算的数据量以及计算本身的代价

**PushdownPredicate**

PushdownPredicate 是最常见的用于减少参与计算的数据量的方法。

直接对两表进行 Join 操作，然后再 进行 Filter 操作。引入 PushdownPredicate 后，可先对两表进行 Filter 再进行 Join

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019013014250169.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW9qaWFvNTIxNzY1MTQ2NTE0,size_16,color_FFFFFF,t_70)

当 Filter 可过滤掉大部分数据时，参与 Join 的数据量大大减少，从而使得 Join 操作速度大大提高。

这里需要说明的是，此处的优化是 LogicalPlan 的优化，从逻辑上保证了将 Filter 下推后由于参与 Join 的数据量变少而提高了性能。另一方面，在物理层面，Filter 下推后，对于支持 Filter 下推的 Storage，并不需要将表的全量数据扫描出来再过滤，而是直接只扫描符合 Filter 条件的数据，从而在物理层面极大减少了扫描表的开销，提高了执行速度。

**ConstantFolding**

本文的 SQL 查询中，Project 部分包含了 100 + 800 + match_score + english_score 。如果不进行优化，那如果有一亿条记录，就会计算一亿次 100 + 80，非常浪费资源。因此可通过 ConstantFolding 将这些常量合并，从而减少不必要的计算，提高执行速度

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190130142719285.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW9qaWFvNTIxNzY1MTQ2NTE0,size_16,color_FFFFFF,t_70)



**ColumnPruning**

在上图中，Filter 与 Join 操作会保留两边所有字段，然后在 Project 操作中筛选出需要的特定列。如果能将 Project 下推，在扫描表时就只筛选出满足后续操作的最小字段集，则能大大减少 Filter 与 Project 操作的中间结果集数据量，从而极大提高执行速度。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190130143128524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW9qaWFvNTIxNzY1MTQ2NTE0,size_16,color_FFFFFF,t_70)

此处的优化是逻辑上的优化。在物理上，Project 下推后，对于列式存储，如 Parquet 和 ORC，可在扫描表时就只扫描需要的列而跳过不需要的列，进一步减少了扫描开销，提高了执行速度。

## 2.4 SparkPlanner

得到优化后的 LogicalPlan 后，SparkPlanner 将其转化为 SparkPlan 即物理计划。

### 第一步:planner.plan(ReturnAnswer(optimizedPlan)).next()

源码地址：org.apache.spark.sql.catalyst.planning.QueryPlanner.scala

```scala
abstract class GenericStrategy[PhysicalPlan <: TreeNode[PhysicalPlan]] extends Logging {

  /**
   * Returns a placeholder for a physical plan that executes `plan`. This placeholder will be
   * filled in automatically by the QueryPlanner using the other execution strategies that are
   * available.
   */
  protected def planLater(plan: LogicalPlan): PhysicalPlan

  def apply(plan: LogicalPlan): Seq[PhysicalPlan]
}


abstract class QueryPlanner[PhysicalPlan <: TreeNode[PhysicalPlan]] {
  /** A list of execution strategies that can be used by the planner */
  def strategies: Seq[GenericStrategy[PhysicalPlan]]

  /**
   * 整合所有的Strategy，_(plan)每个Strategy应用plan上，
   * 得到所有Strategies执行完后生成的所有Physical Plan的集合。
   */
  def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
    // Obviously a lot to do here still...

    // Collect physical plan candidates.
    // strategies 实现了转化
    val candidates = strategies.iterator.flatMap(_(plan))

    // The candidates may contain placeholders marked as [[planLater]],
    // so try to replace them by their child plans.
    // 移除planLater
    val plans = candidates.flatMap { candidate =>
      val placeholders = collectPlaceholders(candidate)

      if (placeholders.isEmpty) {
        // Take the candidate as is because it does not contain placeholders.
        Iterator(candidate)
      } else {
        // Plan the logical plan marked as [[planLater]] and replace the placeholders.
        placeholders.iterator.foldLeft(Iterator(candidate)) {
          case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
            // Plan the logical plan for the placeholder.
            val childPlans = this.plan(logicalPlan)

            candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
              childPlans.map { childPlan =>
                // Replace the placeholder by the child plan
                candidateWithPlaceholders.transformUp {
                  case p if p.eq(placeholder) => childPlan
                }
              }
            }
        }
      }
    }
    // 裁剪plan，去掉 bad plan，但目前只是原封不动返回
    val pruned = prunePlans(plans)
    assert(pruned.hasNext, s"No plan for $plan")
    pruned
  }
```



### 第二步:SparkPlanner继承于SparkStrategies，而SparkStrategies继承了QueryPlanner

```scala
//包含不同策略的策略来优化物理执行计划
class SparkPlanner(
    val sparkContext: SparkContext,
    val conf: SQLConf,
    val experimentalMethods: ExperimentalMethods)
  extends SparkStrategies {

  //partitions的个数 
  def numPartitions: Int = conf.numShufflePartitions

  //策略的集合
  override def strategies: Seq[Strategy] =
    experimentalMethods.extraStrategies ++
      extraPlanningStrategies ++ (
      PythonEvals ::
      DataSourceV2Strategy ::
      //一个针对Hadoop文件系统做的策略，当执行计划的底层Relation是HadoopFsRelation时会调用到，用来扫描文件
      FileSourceStrategy ::
      //Spark针对DataSource预定义了四种scan接口，TableScan、PrunedScan、PrunedFilteredScan、CatalystScan
      //(CatalystScan是unstable的，也是不常用的)，如果开发者（用户）自己实现的DataSource是实现了这四种接口之一的
      //在scan到执行计划的底层Relation时，就会调用来扫描文件
      DataSourceStrategy(conf) ::
      //在Spark SQL中加limit n时候回调用到（如果不指定，Spark 默认也会limit 20），
      //在源码中，会给每种case的limit节点的子节点使用PlanLater
      SpecialLimits ::
      //执行聚合函数的策略
      Aggregation ::
      Window ::
      //JoinSelection用到了相关的统计信息来选择将Join转换为
      //BroadcastHashJoinExec还是ShuffledHashJoinExec
      //还是SortMergeJoinExec，属于CBO基于代价的策略
      JoinSelection ::
      //当数据在内存中被缓存过，就会用到该策略。
      InMemoryScans ::
      //一些基本操作的执行策略，如flatMap，sort，project等，但是实际上大都是给这些节点的子节点套上一个PlanLater。
      BasicOperators :: Nil)

......

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190130175726850.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW9qaWFvNTIxNzY1MTQ2NTE0,size_16,color_FFFFFF,t_70)

直接使用 BroadcastExchangeExec 将数据广播出去，然后结合广播数据对 people 表使用 BroadcastHashJoinExec 进行 Join。再经过 Project 后使用 HashAggregateExec 进行分组聚合。

### 第三步:lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan) 执行前的一些准备工作

```scala
  protected def prepareForExecution(plan: SparkPlan): SparkPlan = {
    //将规则遍历应用到plan
    preparations.foldLeft(plan) { case (sp, rule) => rule.apply(sp) }
  }

  /** A sequence of rules that will be applied in order to the physical plan before execution. */
  protected def preparations: Seq[Rule[SparkPlan]] = Seq(
    PlanSubqueries(sparkSession),
    EnsureRequirements(sparkSession.sessionState.conf),
    CollapseCodegenStages(sparkSession.sessionState.conf),
    ReuseExchange(sparkSession.sessionState.conf),
    ReuseSubquery(sparkSession.sessionState.conf))
```

### 第四步:EnsureRequirements的apply 方法

```scala
  def apply(plan: SparkPlan): SparkPlan = plan.transformUp {
    // TODO: remove this after we create a physical operator for `RepartitionByExpression`.
    case operator @ ShuffleExchangeExec(upper: HashPartitioning, child, _) =>
      child.outputPartitioning match {
        case lower: HashPartitioning if upper.semanticEquals(lower) => child
        case _ => operator
      }
    case operator: SparkPlan =>
      //执行 ensureDistributionAndOrdering
      ensureDistributionAndOrdering(reorderJoinPredicates(operator))
  }
```

### 第五步:ensureDistributionAndOrdering方法

```scala
 /**
   * Preparations 会比较 children的实际输出分布和需求输出分布的不同(比较partitioning)，
   * 然后添加ShuffleExchangeExec、ExchangeCoordinator、SortExec做转化处理，使其匹配
   */
  private def ensureDistributionAndOrdering(operator: SparkPlan): SparkPlan = {
    val requiredChildDistributions: Seq[Distribution] = operator.requiredChildDistribution
    val requiredChildOrderings: Seq[Seq[SortOrder]] = operator.requiredChildOrdering
    var children: Seq[SparkPlan] = operator.children
    assert(requiredChildDistributions.length == children.length)
    assert(requiredChildOrderings.length == children.length)

    // Ensure that the operator's children satisfy their output distribution requirements.
    // children的实际输出分布(其实就是partitioning)满足要求的输出分布
    children = children.zip(requiredChildDistributions).map {
      //满足，直接返回
      case (child, distribution) if child.outputPartitioning.satisfies(distribution) =>
        child
      // 广播单独处理
      case (child, BroadcastDistribution(mode)) =>
        BroadcastExchangeExec(mode, child)
      case (child, distribution) =>
        val numPartitions = distribution.requiredNumPartitions
          .getOrElse(defaultNumPreShufflePartitions)
         // 做一次shuffle(可以认为是重新分区,改变分布)，满足需求。
        ShuffleExchangeExec(distribution.createPartitioning(numPartitions), child)
    }

    // Get the indexes of children which have specified distribution requirements and need to have
    // same number of partitions.
    // 过滤有指定分布需求的children
    val childrenIndexes = requiredChildDistributions.zipWithIndex.filter {
      case (UnspecifiedDistribution, _) => false
      case (_: BroadcastDistribution, _) => false
      case _ => true
    }.map(_._2)

    // children 的 partition 数量
    val childrenNumPartitions =
      childrenIndexes.map(children(_).outputPartitioning.numPartitions).toSet

    if (childrenNumPartitions.size > 1) {
      // Get the number of partitions which is explicitly required by the distributions.
      // children 的分布需要的partition数量
      val requiredNumPartitions = {
        val numPartitionsSet = childrenIndexes.flatMap {
          index => requiredChildDistributions(index).requiredNumPartitions
        }.toSet
        assert(numPartitionsSet.size <= 1,
          s"$operator have incompatible requirements of the number of partitions for its children")
        numPartitionsSet.headOption
      }
      // partition 数量目标
      val targetNumPartitions = requiredNumPartitions.getOrElse(childrenNumPartitions.max)
      // 指定分布的children的partition数量全部统一为 targetNumPartitions
      children = children.zip(requiredChildDistributions).zipWithIndex.map {
        case ((child, distribution), index) if childrenIndexes.contains(index) =>
          if (child.outputPartitioning.numPartitions == targetNumPartitions) {
            child   // 符合目标，直接返回
          } else {
            // 不符合目标，shuffle
            val defaultPartitioning = distribution.createPartitioning(targetNumPartitions)
            child match {
              // If child is an exchange, we replace it with a new one having defaultPartitioning.
              case ShuffleExchangeExec(_, c, _) => ShuffleExchangeExec(defaultPartitioning, c)
              case _ => ShuffleExchangeExec(defaultPartitioning, child)
            }
          }

        case ((child, _), _) => child
      }
    }

    // Now, we need to add ExchangeCoordinator if necessary.
    // Actually, it is not a good idea to add ExchangeCoordinators while we are adding Exchanges.
    // However, with the way that we plan the query, we do not have a place where we have a
    // global picture of all shuffle dependencies of a post-shuffle stage. So, we add coordinator
    // at here for now.
    // Once we finish https://issues.apache.org/jira/browse/SPARK-10665,
    // we can first add Exchanges and then add coordinator once we have a DAG of query fragments.
    //添加ExchangeCoordinator,调节多个spark plan的数据分布
    children = withExchangeCoordinator(children, requiredChildDistributions)

    // Now that we've performed any necessary shuffles, add sorts to guarantee output orderings:
    // 如果有sort的需求，则加上SortExec
    children = children.zip(requiredChildOrderings).map { case (child, requiredOrdering) =>
      // If child.outputOrdering already satisfies the requiredOrdering, we do not need to sort.
      if (SortOrder.orderingSatisfies(child.outputOrdering, requiredOrdering)) {
        child
      } else {
        SortExec(requiredOrdering, global = false, child = child)
      }
    }

    operator.withNewChildren(children)
  }
```

### 第六步:withExchangeCoordinator(children, requiredChildDistributions)方法

```scala
  private def withExchangeCoordinator(
      children: Seq[SparkPlan],
      requiredChildDistributions: Seq[Distribution]): Seq[SparkPlan] = {
    // 判断是否需要添加 ExchangeCoordinator
    val supportsCoordinator =
      if (children.exists(_.isInstanceOf[ShuffleExchangeExec])) {
        // Right now, ExchangeCoordinator only support HashPartitionings.
        children.forall {
          // 条件1 children中有ShuffleExchangeExec且分区为HashPartitioning  
          case e @ ShuffleExchangeExec(hash: HashPartitioning, _, _) => true
          case child =>
            child.outputPartitioning match {
              case hash: HashPartitioning => true
              case collection: PartitioningCollection =>
                collection.partitionings.forall(_.isInstanceOf[HashPartitioning])
              case _ => false
            }
        }
      } else {
        // In this case, although we do not have Exchange operators, we may still need to
        // shuffle data when we have more than one children because data generated by
        // these children may not be partitioned in the same way.
        // Please see the comment in withCoordinator for more details.
        // 条件2 分布为ClusteredDistribution 或 HashClusteredDistribution
        val supportsDistribution = requiredChildDistributions.forall { dist =>
          dist.isInstanceOf[ClusteredDistribution] || dist.isInstanceOf[HashClusteredDistribution]
        }
        children.length > 1 && supportsDistribution
      }

    val withCoordinator =
      // adaptiveExecutionEnabled 且 符合条件1或2
      //adaptiveExecutionEnabled 默认是false的，所以ExchangeCoordinator默认是关闭的
      if (adaptiveExecutionEnabled && supportsCoordinator) {
        val coordinator =
          new ExchangeCoordinator(
            targetPostShuffleInputSize,
            minNumPostShufflePartitions)
        children.zip(requiredChildDistributions).map {
          case (e: ShuffleExchangeExec, _) =>
            // This child is an Exchange, we need to add the coordinator.
            // 条件1， 直接添加 coordinator
            e.copy(coordinator = Some(coordinator))
          case (child, distribution) =>
            // If this child is not an Exchange, we need to add an Exchange for now.
            // Ideally, we can try to avoid this Exchange. However, when we reach here,
            // there are at least two children operators (because if there is a single child
            // and we can avoid Exchange, supportsCoordinator will be false and we
            // will not reach here.). Although we can make two children have the same number of
            // post-shuffle partitions. Their numbers of pre-shuffle partitions may be different.
            // For example, let's say we have the following plan
            //         Join
            //         /  \
            //       Agg  Exchange
            //       /      \
            //    Exchange  t2
            //      /
            //     t1
            // In this case, because a post-shuffle partition can include multiple pre-shuffle
            // partitions, a HashPartitioning will not be strictly partitioned by the hashcodes
            // after shuffle. So, even we can use the child Exchange operator of the Join to
            // have a number of post-shuffle partitions that matches the number of partitions of
            // Agg, we cannot say these two children are partitioned in the same way.
            // Here is another case
            //         Join
            //         /  \
            //       Agg1  Agg2
            //       /      \
            //   Exchange1  Exchange2
            //       /       \
            //      t1       t2
            // In this case, two Aggs shuffle data with the same column of the join condition.
            // After we use ExchangeCoordinator, these two Aggs may not be partitioned in the same
            // way. Let's say that Agg1 and Agg2 both have 5 pre-shuffle partitions and 2
            // post-shuffle partitions. It is possible that Agg1 fetches those pre-shuffle
            // partitions by using a partitionStartIndices [0, 3]. However, Agg2 may fetch its
            // pre-shuffle partitions by using another partitionStartIndices [0, 4].
            // So, Agg1 and Agg2 are actually not co-partitioned.
            //
            // It will be great to introduce a new Partitioning to represent the post-shuffle
            // partitions when one post-shuffle partition includes multiple pre-shuffle partitions.
            // 条件2，这个child的children虽然分区数目一样，但是不一定是同一种分区方式，所以加上coordinator
            val targetPartitioning = distribution.createPartitioning(defaultNumPreShufflePartitions)
            assert(targetPartitioning.isInstanceOf[HashPartitioning])
            ShuffleExchangeExec(targetPartitioning, child, Some(coordinator))
        }
      } else {
        // If we do not need ExchangeCoordinator, the original children are returned.
        children
      }

    withCoordinator
  }
```

### 第七步:executedPlan.execute()最后一步执行

```scala
  final def execute(): RDD[InternalRow] = executeQuery {
    if (isCanonicalizedPlan) {
      throw new IllegalStateException("A canonicalized plan is not supposed to be executed.")
    }
    //执行各个具体SparkPlan的doExecute函数
    doExecute()
  }
```

```scala
  protected final def executeQuery[T](query: => T): T = {
    RDDOperationScope.withScope(sparkContext, nodeName, false, true) {
      prepare()
      waitForSubqueries()
      query
    }
  }
```

```scala
  final def prepare(): Unit = {
    // doPrepare() may depend on it's children, we should call prepare() on all the children first.
    children.foreach(_.prepare())
    synchronized {
      if (!prepared) {
        prepareSubqueries()
        doPrepare()
        prepared = true
      }
    }
  }
```

关键是两个函数doPrepare和doExecute方法

### 第八步:以排序为例看一下SortExec的 doExecute方法

```scala
  protected override def doExecute(): RDD[InternalRow] = {
    val peakMemory = longMetric("peakMemory")
    val spillSize = longMetric("spillSize")
    val sortTime = longMetric("sortTime")

    //调用child的execute方法，然后对每个partition进行排序
    child.execute().mapPartitionsInternal { iter =>
      val sorter = createSorter()

      val metrics = TaskContext.get().taskMetrics()
      // Remember spill data size of this task before execute this operator so that we can
      // figure out how many bytes we spilled for this operator.
      val spillSizeBefore = metrics.memoryBytesSpilled
      // 排序
      val sortedIterator = sorter.sort(iter.asInstanceOf[Iterator[UnsafeRow]])
      sortTime += sorter.getSortTimeNanos / 1000000
      peakMemory += sorter.getPeakMemoryUsage
      spillSize += metrics.memoryBytesSpilled - spillSizeBefore
      metrics.incPeakExecutionMemory(sorter.getPeakMemoryUsage)

      sortedIterator
    }
  }
```

### 第九步 :child.execute() 也就是ShuffleExchangeExec

```scala
  private[this] val exchanges = ArrayBuffer[ShuffleExchangeExec]()

  override protected def doPrepare(): Unit = {
    // If an ExchangeCoordinator is needed, we register this Exchange operator
    // to the coordinator when we do prepare. It is important to make sure
    // we register this operator right before the execution instead of register it
    // in the constructor because it is possible that we create new instances of
    // Exchange operators when we transform the physical plan
    // (then the ExchangeCoordinator will hold references of unneeded Exchanges).
    // So, we should only call registerExchange just before we start to execute
    // the plan.
    coordinator match {
      // 向exchangeCoordinator注册该exchange
      case Some(exchangeCoordinator) => exchangeCoordinator.registerExchange(this)
      case _ =>
    }
  }

  @GuardedBy("this")
  // 注册就是添加到array中
  def registerExchange(exchange: ShuffleExchangeExec): Unit = synchronized {
    exchanges += exchange
  }

```

```scala
  protected override def doExecute(): RDD[InternalRow] = attachTree(this, "execute") {
    // Returns the same ShuffleRowRDD if this plan is used by multiple plans.
     // 有缓存则直接返回缓存
    if (cachedShuffleRDD == null) {
      cachedShuffleRDD = coordinator match {
      // 有exchangeCoordinator  
      case Some(exchangeCoordinator) =>
          val shuffleRDD = exchangeCoordinator.postShuffleRDD(this)
          assert(shuffleRDD.partitions.length == newPartitioning.numPartitions)
          shuffleRDD
       // 没有exchangeCoordinator
        case _ =>
          val shuffleDependency = prepareShuffleDependency()
          preparePostShuffleRDD(shuffleDependency)
      }
    }
    cachedShuffleRDD
  }
```



### 第十步:没有exchangeCoordinator的情况 prepareShuffleDependency()方法

```scala
  /**
   * 返回一个ShuffleDependency，ShuffleDependency中最重要的是rddWithPartitionIds，
   * 它决定了每一条InternalRow shuffle后的partition id
   */
  private[exchange] def prepareShuffleDependency()
    : ShuffleDependency[Int, InternalRow, InternalRow] = {
    ShuffleExchangeExec.prepareShuffleDependency(
      child.execute(), child.output, newPartitioning, serializer)
  }
```

```scala
  def prepareShuffleDependency(
      rdd: RDD[InternalRow],
      outputAttributes: Seq[Attribute],
      newPartitioning: Partitioning,
      serializer: Serializer): ShuffleDependency[Int, InternalRow, InternalRow] = {
 
  ......

    // Now, we manually create a ShuffleDependency. Because pairs in rddWithPartitionIds
    // are in the form of (partitionId, row) and every partitionId is in the expected range
    // [0, part.numPartitions - 1]. The partitioner of this is a PartitionIdPassthrough.
    val dependency =
      new ShuffleDependency[Int, InternalRow, InternalRow](
        rddWithPartitionIds,
        new PartitionIdPassthrough(part.numPartitions),
        serializer)

    dependency
  }
```

### 第十一步: preparePostShuffleRDD(shuffleDependency)方法

```scala
  private[exchange] def preparePostShuffleRDD(
      shuffleDependency: ShuffleDependency[Int, InternalRow, InternalRow],
      specifiedPartitionStartIndices: Option[Array[Int]] = None): ShuffledRowRDD = {
    // If an array of partition start indices is provided, we need to use this array
    // to create the ShuffledRowRDD. Also, we need to update newPartitioning to
    // update the number of post-shuffle partitions.
    // 如果specifiedPartitionStartIndices存在，它将决定shuffle后的分区情况
    // exchangeCoordinator 会用到specifiedPartitionStartIndices来实现功能
    specifiedPartitionStartIndices.foreach { indices =>
      assert(newPartitioning.isInstanceOf[HashPartitioning])
      newPartitioning = UnknownPartitioning(indices.length)
    }
    new ShuffledRowRDD(shuffleDependency, specifiedPartitionStartIndices)
  }
```

```scala
//返回结果是ShuffledRowRDD
class ShuffledRowRDD(
    var dependency: ShuffleDependency[Int, InternalRow, InternalRow],
    specifiedPartitionStartIndices: Option[Array[Int]] = None)
  extends RDD[InternalRow](dependency.rdd.context, Nil) {

  // 分区数目
  private[this] val numPreShufflePartitions = dependency.partitioner.numPartitions

  // 每个partition的startIndice
  private[this] val partitionStartIndices: Array[Int] = specifiedPartitionStartIndices match {
    case Some(indices) => indices
    case None =>
      // When specifiedPartitionStartIndices is not defined, every post-shuffle partition
      // corresponds to a pre-shuffle partition.
      (0 until numPreShufflePartitions).toArray
  }

  // rdd 的partitioner
  private[this] val part: Partitioner =
    new CoalescedPartitioner(dependency.partitioner, partitionStartIndices)

  override def getDependencies: Seq[Dependency[_]] = List(dependency)

  override val partitioner: Option[Partitioner] = Some(part)

  // 获取所有的partition
  override def getPartitions: Array[Partition] = {
    assert(partitionStartIndices.length == part.numPartitions)
    Array.tabulate[Partition](partitionStartIndices.length) { i =>
      val startIndex = partitionStartIndices(i)
      val endIndex =
        if (i < partitionStartIndices.length - 1) {
          partitionStartIndices(i + 1)
        } else {
          numPreShufflePartitions
        }
      new ShuffledRowRDDPartition(i, startIndex, endIndex)
    }
  }

  override def getPreferredLocations(partition: Partition): Seq[String] = {
    val tracker = SparkEnv.get.mapOutputTracker.asInstanceOf[MapOutputTrackerMaster]
    val dep = dependencies.head.asInstanceOf[ShuffleDependency[_, _, _]]
    tracker.getPreferredLocationsForShuffle(dep, partition.index)
  }

  override def compute(split: Partition, context: TaskContext): Iterator[InternalRow] = {
    val shuffledRowPartition = split.asInstanceOf[ShuffledRowRDDPartition]
    // The range of pre-shuffle partitions that we are fetching at here is
    // [startPreShufflePartitionIndex, endPreShufflePartitionIndex - 1].
    val reader =
      SparkEnv.get.shuffleManager.getReader(
        dependency.shuffleHandle,
        shuffledRowPartition.startPreShufflePartitionIndex,
        shuffledRowPartition.endPreShufflePartitionIndex,
        context)
    reader.read().asInstanceOf[Iterator[Product2[Int, InternalRow]]].map(_._2)
  }

  override def clearDependencies() {
    super.clearDependencies()
    dependency = null
  }
}
```

```scala
/**
 * A Partitioner that might group together one or more partitions from the parent.
 *
 * @param parent a parent partitioner
 * @param partitionStartIndices indices of partitions in parent that should create new partitions
 *   in child (this should be an array of increasing partition IDs). For example, if we have a
 *   parent with 5 partitions, and partitionStartIndices is [0, 2, 4], we get three output
 *   partitions, corresponding to partition ranges [0, 1], [2, 3] and [4] of the parent partitioner.
 */
class CoalescedPartitioner(val parent: Partitioner, val partitionStartIndices: Array[Int])
  extends Partitioner {
  // 实现 partition 的转换 
  @transient private lazy val parentPartitionMapping: Array[Int] = {
    val n = parent.numPartitions
    val result = new Array[Int](n)
    for (i <- 0 until partitionStartIndices.length) {
      val start = partitionStartIndices(i)
      val end = if (i < partitionStartIndices.length - 1) partitionStartIndices(i + 1) else n
      for (j <- start until end) {
        result(j) = i
      }
    }
    result
  }
```

### 第十二步: 有exchangeCoordinator的情况 exchangeCoordinator.postShuffleRDD(this)方法

```scala
//返回的是ShuffledRowRDD
  def postShuffleRDD(exchange: ShuffleExchangeExec): ShuffledRowRDD = {
    doEstimationIfNecessary()

    if (!postShuffleRDDs.containsKey(exchange)) {
      throw new IllegalStateException(
        s"The given $exchange is not registered in this coordinator.")
    }

    postShuffleRDDs.get(exchange)
  }
```

```scala
  @GuardedBy("this")
  private def doEstimationIfNecessary(): Unit = synchronized {
    // It is unlikely that this method will be called from multiple threads
    // (when multiple threads trigger the execution of THIS physical)
    // because in common use cases, we will create new physical plan after
    // users apply operations (e.g. projection) to an existing DataFrame.
    // However, if it happens, we have synchronized to make sure only one
    // thread will trigger the job submission.
    if (!estimated) {
      // Make sure we have the expected number of registered Exchange operators.
      assert(exchanges.length == numExchanges)

      val newPostShuffleRDDs = new JHashMap[ShuffleExchangeExec, ShuffledRowRDD](numExchanges)

      // Submit all map stages
      val shuffleDependencies = ArrayBuffer[ShuffleDependency[Int, InternalRow, InternalRow]]()
      val submittedStageFutures = ArrayBuffer[SimpleFutureAction[MapOutputStatistics]]()
      var i = 0
      // 依次执行每个注册的exchange的prepareShuffleDependency方法
      while (i < numExchanges) {
        val exchange = exchanges(i)
        val shuffleDependency = exchange.prepareShuffleDependency()
        shuffleDependencies += shuffleDependency
        if (shuffleDependency.rdd.partitions.length != 0) {
          // submitMapStage does not accept RDD with 0 partition.
          // So, we will not submit this dependency.
          submittedStageFutures +=
            exchange.sqlContext.sparkContext.submitMapStage(shuffleDependency)
        }
        i += 1
      }

      // Wait for the finishes of those submitted map stages.
      // 统计结果
      val mapOutputStatistics = new Array[MapOutputStatistics](submittedStageFutures.length)
      var j = 0
      while (j < submittedStageFutures.length) {
        // This call is a blocking call. If the stage has not finished, we will wait at here.
        mapOutputStatistics(j) = submittedStageFutures(j).get()
        j += 1
      }

      // If we have mapOutputStatistics.length < numExchange, it is because we do not submit
      // a stage when the number of partitions of this dependency is 0.
      assert(mapOutputStatistics.length <= numExchanges)

      // Now, we estimate partitionStartIndices. partitionStartIndices.length will be the
      // number of post-shuffle partitions.
      // 得到partitionStartIndices
      val partitionStartIndices =
        if (mapOutputStatistics.length == 0) {
          Array.empty[Int]
        } else {
          // 根据 mapOutputStatistics 获取 partitionStartIndices
          estimatePartitionStartIndices(mapOutputStatistics)
        }
      // 执行preparePostShuffleRDD，和没有exchangeCoordinator唯一的不同是有partitionStartIndices参数！
      var k = 0
      while (k < numExchanges) {
        val exchange = exchanges(k)
        val rdd =
          exchange.preparePostShuffleRDD(shuffleDependencies(k), Some(partitionStartIndices))
        newPostShuffleRDDs.put(exchange, rdd)

        k += 1
      }

      // Finally, we set postShuffleRDDs and estimated.
      assert(postShuffleRDDs.isEmpty)
      assert(newPostShuffleRDDs.size() == numExchanges)
      // 结果放入缓存
      postShuffleRDDs.putAll(newPostShuffleRDDs)
      estimated = true
    }
  }
```

```scala
  /**
   * Estimates partition start indices for post-shuffle partitions based on
   * mapOutputStatistics provided by all pre-shuffle stages.
   */
  def estimatePartitionStartIndices(
      mapOutputStatistics: Array[MapOutputStatistics]): Array[Int] = {
    // If minNumPostShufflePartitions is defined, it is possible that we need to use a
    // value less than advisoryTargetPostShuffleInputSize as the target input size of
    // a post shuffle task.
    // 每个partition的目标inputsize，即每个分区数据量的大小
    val targetPostShuffleInputSize = minNumPostShufflePartitions match {
      case Some(numPartitions) =>
        val totalPostShuffleInputSize = mapOutputStatistics.map(_.bytesByPartitionId.sum).sum
        // The max at here is to make sure that when we have an empty table, we
        // only have a single post-shuffle partition.
        // There is no particular reason that we pick 16. We just need a number to
        // prevent maxPostShuffleInputSize from being set to 0.
        val maxPostShuffleInputSize =
          math.max(math.ceil(totalPostShuffleInputSize / numPartitions.toDouble).toLong, 16)
        math.min(maxPostShuffleInputSize, advisoryTargetPostShuffleInputSize)

      case None => advisoryTargetPostShuffleInputSize
    }

    logInfo(
      s"advisoryTargetPostShuffleInputSize: $advisoryTargetPostShuffleInputSize, " +
      s"targetPostShuffleInputSize $targetPostShuffleInputSize.")

    // Make sure we do get the same number of pre-shuffle partitions for those stages.
    // 得到分区数，应该有且只有一个数值
    val distinctNumPreShufflePartitions =
      mapOutputStatistics.map(stats => stats.bytesByPartitionId.length).distinct
    // The reason that we are expecting a single value of the number of pre-shuffle partitions
    // is that when we add Exchanges, we set the number of pre-shuffle partitions
    // (i.e. map output partitions) using a static setting, which is the value of
    // spark.sql.shuffle.partitions. Even if two input RDDs are having different
    // number of partitions, they will have the same number of pre-shuffle partitions
    // (i.e. map output partitions).
    assert(
      distinctNumPreShufflePartitions.length == 1,
      "There should be only one distinct value of the number pre-shuffle partitions " +
        "among registered Exchange operator.")
    val numPreShufflePartitions = distinctNumPreShufflePartitions.head
     // 开始构建partitionStartIndices  
    val partitionStartIndices = ArrayBuffer[Int]()
    // The first element of partitionStartIndices is always 0.
    partitionStartIndices += 0

    var postShuffleInputSize = 0L
    // 根据targetPostShuffleInputSize，对分区进行调整，会做一些合并之类的操作。
    var i = 0
    while (i < numPreShufflePartitions) {
      // We calculate the total size of ith pre-shuffle partitions from all pre-shuffle stages.
      // Then, we add the total size to postShuffleInputSize.
      var nextShuffleInputSize = 0L
      var j = 0
      while (j < mapOutputStatistics.length) {
        nextShuffleInputSize += mapOutputStatistics(j).bytesByPartitionId(i)
        j += 1
      }

      // If including the nextShuffleInputSize would exceed the target partition size, then start a
      // new partition.
      if (i > 0 && postShuffleInputSize + nextShuffleInputSize > targetPostShuffleInputSize) {
        partitionStartIndices += i
        // reset postShuffleInputSize.
        postShuffleInputSize = nextShuffleInputSize
      } else postShuffleInputSize += nextShuffleInputSize

      i += 1
    }

    partitionStartIndices.toArray
  }
```

有exchangeCoordinator的情况就生成了partitionStartIndices，从而对分区进行了调整

execute 最终的输出是rdd，剩下的结果便是spark对rdd的运算了。其实 spark sql 最终的目标便也是生成rdd，交给spark core来运算

**本文参考:**

```http
https://blog.csdn.net/jiaojiao521765146514/article/details/86627052
```



