---
layout: post
title: Spark SQL SqlPaser Semantic Parsing
date: 2014-08-23
categories: spark
---

Spark SQL是spark1.0.0版本里的新功能，其实对于绝大多数的项目来说，spark sql轻松地集成了shark，hive等小项目的功能，说明了spark的可扩展性极高（RDD数据集的设计特性与数据库列表结构完美契合）。现在的master版本正在将Hive-thriftserver与JDBC集成进入Spark SQL，可以预见在未来底层数据库与spark将会实现完美对接。

扯了这么久，还是说下这篇博客的主题，其实也算是对于自己之前部分工作的总结。主要是自己修改spark源代码，解决spark的jira上issue-2096。这个问题是关于sql语句里nested structure解析，对于JsonRDD来说，其Schema中存有arrayofstruct的问题。

arrayofstruct的问题类似这样：

    {"id" :1, "name": "king", "scores":[{"math":90, "phy":80, "che": 88}, {"math":98, "phy":96, "che": 94}, {"math":93, "phy":89, "che": 82}] }
    {"id" :2, "name": "joe", "scores":[{"math":92, "phy":82, "che": 90}, {"math":100, "phy":93, "che": 91}, {"math":90, "phy":86, "che": 79}] }

上面的两行json中的scores（多次考试成绩）就是arrayofstruct。

如此现在我需要获取scores.math（也就是每次数学考试成绩）数据，或者scores[0].math（第一次数学考试成绩），spark sql就会error，无法解析。

此处给出两个链接：

 + jira的issue地址：           [https://issues.apache.org/jira/browse/SPARK-2096](https://issues.apache.org/jira/browse/SPARK-2096)

 + github的pull request地址：  [https://github.com/apache/spark/pull/2082](https://github.com/apache/spark/pull/2082)

在解决上述issue的过程中，自己主要涉及到sql解析的工作，也就是spark sql里SqlPaser类，还有LogicalPlan。

关于这个类的一些说明网络上也都有，这篇博客的博主写的比较完善，但缺少一些关键的深层次解析（http://blog.csdn.net/oopsoom/article/details/37943507），我不会再重复小伙伴的内容，所以建议先阅读完这篇博客再看我的文章。 所以我做的就是在该小伙伴（好像是我朋友的朋友的朋友）基础上再添加一二，也就是对于上述两个类的分析。

---

### SqlPaser

首先，你得有个spark源代码IDE环境。Intellij Idea中以Maven或者Sbt两种方式导入spark-master版本源代码（这样一句话带过好像有点不厚道，因为这过程总会出现奇怪的问题）。examples总是跑不起来，直接进入sql-core-src-test-scala-org.apache.spark.sql-json，用JsonSuite开始跟踪调试即可。

    test("Complex field and type inferring")

    checkAnswer(
          sql("select arrayOfString[0], arrayOfString[1], arrayOfString[2] from jsonTable"),
          ("str1", "str2", null) :: Nil
        )

在上面的sql语句处设置一个断点。我们可以进入sql语句跟踪。然后接着（要不要先列出所有相关的断点位置？）

    SqlPaser.scala - phrase

    phrase(query)(new lexical.Scanner(input)) match {
            case Success(r, x) => r
            case x => sys.error(x.toString)
          }

此处最难理解的就是 phrase(query)(new lexical.Scanner(input)),好吧，我一开始也是比较痛苦的想办法理解phrase, lexical, Scanner这些词。实际上这些是scala.util.parsing.combinator包里的文件，是scala语言对于字符串或者其他需要解析的内容的函数库包。需要补充说明的是，对于这个函数库包各个文件的作用，为了便于理解，可以继续在以下两处设置breakpoint。

    Scanner.scala - new Scanner

    def first = tok
    def rest = new Scanner(rest2)
    def pos = rest1.pos

此处Scanner就像不断往后scan这个输入的字符串，根据Token识别，根据whitespace消除空格。所以我们可以看看Token是如何定义的。此处假设你已经看明白~ ^^ 等符号作用。

    SqlPaser.scala - case first, case i

    override lazy val token: Parser[Token] = (
      identChar ~ rep( identChar | digit ) ^^
      {
        case first ~ rest if(first != '.') => processIdent(first :: rest mkString "")
        case first ~ rest if(first == '.') => StringLit(rest mkString "")
      }
      | rep1(digit) ~ opt('.' ~> rep(digit)) ^^ {
      case i ~ None    => NumericLit(i mkString "")
      case i ~ Some(d) => FloatLit(i.mkString("") + "." + d.mkString(""))
      }


此处解释下：identChar ~ rep(identChar \| digit) ^^ 。意思就是以identChar（字母，下划线_，dot.符号三者）开头的，后续为重复字符或数字的Token。此处插入一句划分Token的为分界符delimiters，所以我们看到arrayOfString\[0\]时，\[ 作为分界符出现，也就是Token为arrayOfString。当这样的Token出现或者满足（通过Scanner）时，进入到^^后{}包含的函数内，函数内出现case，也就是满足条件first ~ rest（与前面Token格式组成对应），后续if判断此Token是否以dot.开头，此处为我为了arrayOfStruct\[0\].field1解析而做的修改。将field1识别为StringLit。\|执行顺序为前一项不满足时执行后者，若满足则不再执行，所以\|这个顺序也颇有意思，在前面有个lazy val baseExpression就也涉及，baseExpression是我们后续分析的一处重点。

跟踪到此处时就会发现Scanner与Token的关系（在Scanner类里有具体实现），同时也会了解各符号作用，还有SqlLexical extends StdLexical，根据一部字典（Lexical）解析sql语句，分界符是delimiters，关键字是符合Token定义规则的通过分界符筛选出来的Token。

然后回到phrase，这个函数的定义是将传入query的Scanner获得的Token用PackratReader包装，我这个Scala半调子水平也是看了好久才看明白为啥phrase有两个参数传递，因为Scanner作为参数传给了def apply(in: Input)。

    /**
       *  A parser generator delimiting whole phrases (i.e. programs).
       *
       *  Overridden to make sure any input passed to the argument parser
       *  is wrapped in a `PackratReader`.
       */
      override def phrase[T](p: Parser[T]) = {
        val q = super.phrase(p)
        new PackratParser[T] {
          def apply(in: Input) = in match {
            case in: PackratReader[_] => q(in)
            case in => q(new PackratReader(in))
          }
        }
      }

所以我们理解了此处的phrase函数，such a smart design.回到外面继续，所以我们知道query逐个Token解析，例如一句sql: "select arrayOfString\[0\], arrayOfString\[1\], arrayOfString\[2\] from jsonTable"，解析为select arrayOfString 0 arrayOfString 1 arrayOfString 2 from jsonTable 一共9个Token。如果是arrayOfStruct\[0\].field，则对应为arrayOfStruct 0 field三个Token。注意此处dot解析只在遇到前面为分界符，也就是dot开头的情况只会是跟随在分界符后才会出现。

如此我们得到一个对sql语句的初步解析-解析为单个Token。后面就是将这些Token解析生成一棵树。关于这棵树长什么样，前面小伙伴博主的另外一篇博客有详细的说明（http://blog.csdn.net/oopsoom/article/details/38084079）。说白了就是二叉树的结构，下面设置一个断点可以查看整棵树的生成过程。

      SqlPaser - case d ~ p ~ r ~ f ~ g ~ h ~ o ~ l

      protected lazy val select: Parser[LogicalPlan] =
        SELECT ~> opt(DISTINCT) ~ projections ~
        opt(from) ~ opt(filter) ~
        opt(grouping) ~
        opt(having) ~
        opt(orderBy) ~
        opt(limit) <~ opt(";") ^^ {
          case d ~ p ~ r ~ f ~ g ~ h ~ o ~ l  =>
            val base = r.getOrElse(NoRelation)
            val withFilter = f.map(f => Filter(f, base)).getOrElse(base)
            val withProjection =
              g.map {g =>
                Aggregate(assignAliases(g), assignAliases(p), withFilter)
              }.getOrElse(Project(assignAliases(p), withFilter))
            val withDistinct = d.map(_ => Distinct(withProjection)).getOrElse(withProjection)
            val withHaving = h.map(h => Filter(h, withDistinct)).getOrElse(withDistinct)
            val withOrder = o.map(o => Sort(o, withHaving)).getOrElse(withHaving)
            val withLimit = l.map { l => Limit(l, withOrder) }.getOrElse(withOrder)
            withLimit
      }

首先SELECT是Keyword，根据隐函数直接转换为Parser，projections，可以看到返回的是Paser\[Seq\[Expression\]\]，再判断下是否有DISTINCT，from在sql中称为Relation，类型是Paser\[LogicalPlan\]，filter返回Parser\[Expression\]，后续的grouping，having，orderBy，limit均返回Parser\[Expression\]。理解下这也就是我们构建sql语句的正常流程。而其实后续case里的过程更加精彩，是我们一般思维下解析sql语句的过程，而现在这过程就是构建二叉树的过程。这其中我自己也还是有很多不明白为什么这么设计的地方，但其设计结果却是十分令人惊叹。

首先标记下，前面sql语句构建中，只有from返回的是Paser\[LogicalPlan\]，后续每一步都是在这基础上继续添加新的节点，这节点仔细观察会发现：

如果是两个LogicalPlan的Paser合并（select中一般为Relation Join操作），则返回类型具有BinaryNode特性；

如果是一个LogicalPlan与一个Expression合并，则生成为UnaryNode；

而叶子节点本身已经包含在LogicalPlan与Expression中，例如NoRelation，AttributeReference，UnresolvedAttribute（在进行Alias操作时添加，此处也是显得非常巧妙的，将Expression转换为NamedExpression，并且with UnaryNode的特性）

关于Alias必须详细说明下，这里其实是后续分析的重点，因为此处影响到了树结构中的重要特性（Resolved与Unresolved的问题）。

先看这个特性的来源。

一棵树我们选择projections作为跟踪对象，发现由repsep(projection, ",")，到projection，发现是expression通过Alias()函数生成，那这个expression就是我们要跟踪的对象，一直找下去就发现会到baseExpression，设置断点。

    SqlPaser.scala - GetItem(base, ordinal) , ident ^^ UnresolvedAttribute

    protected lazy val baseExpression: PackratParser[Expression] =
        expression ~ "[" ~ expression ~ "]" ~ expression ^^ {
          case base ~ _ ~ ordinal ~ _ ~ field => GetArrayOfStructItem(base, ordinal, field)
        } |
        expression ~ "[" ~ expression <~ "]" ^^ {
          case base ~ _ ~ ordinal => GetItem(base, ordinal)
        } |
        TRUE ^^^ Literal(true, BooleanType) |
        FALSE ^^^ Literal(false, BooleanType) |
        cast |
        "(" ~> expression <~ ")" |
        function |
        "-" ~> literal ^^ UnaryMinus |
        ident ^^ UnresolvedAttribute |
        "*" ^^^ Star(None) |
        literal

此处我添加了修改，主要解决arrayOfStruct\[0\].field1问题（其实也就是为了项目将就解决下）：

    expression ~ "[" ~ expression ~ "]" ~ expression ^^ {
              case base ~ _ ~ ordinal ~ _ ~ field => GetArrayOfStructItem(base, ordinal, field)
            } |

读者可以自己理解为何做这样的修改，继续回到原来代码。

GetItem()是个case class， extends Expression，而且查看GetItem可以看到这是最终的解析，eval函数就是计算Expression的值，通过对于input（一直都是TableSchema，根据json注册生成的表结构）,通过分析比较能够找到路径，就是说明这个Attribute是能够计算的，是存在对应值的，也就是能够resolved的。GetItem已经是能够计算最终值的Expression，因为resolved是lazy val，所以直到execute()，我们才看到resolved变成true，这也说明跟踪过程中，arrayOfString\[0\]在LogicalPlan的resolve中是直接返回（在options.distinct match中，匹配case Seq((a, Nil)) => Some(a)）。这里的思想就是先验证是否能够resolved，在通过eval计算该Expression的值。

此处我们看一个非常巧妙的设计，expression其实是一个递归。终止条件是那些跟expression无关的\|，有expression的都是会继续递归下去的。所以，跟踪的时候我们可以看到arrayOfString\[0\]中，arrayOfString是UnresolvedAttribute，0是Literal（extends LeafExpression），两者返回上一层递归调用（expression ~ "\[" ~ expression <~ "\]" ^^ ）后，组合成一个GetItem()，也就是标记——此处这两个Expression包含在一个GetItem的Expression中一起解析。此处也说明下，源代码中另外一种解析是GetField，在LogicalPlan中调用（我朋友Wei Li添加了GetArrayField，我添加了GetArrayOfStructItem）。

那若是遇到struct.field1这样的情况，其并不是返回一个类似GetItem()的Expression，而直接是一个ident，也就是UnresolvedAttribute， 接着看看ident是如何定义的。

    StdTokenPasers.scala

    elem("identifier", _.isInstanceOf[Identifier]) ^^ (_.chars)

也就是该Token是以identifier开头的，跟着Identifier类型。我们查看Identifier并没有什么有效的信息，再查看Token定义生成的地方，在前面SqlLexical中的Token有一处代码

    case first ~ rest if(first != '.') => processIdent(first :: rest mkString "")

查看processIdent

    StdLexical.scala

    protected def processIdent(name: String) =
        if (reserved contains name) Keyword(name) else Identifier(name)

也就是传入的String不是Keyword的话，就作为一个Identifier。所以struct.field1是一个UnresolvedAttribute，并且没有进入递归。这在后续的LogicalPlan中还会有分析。

此处另外有个问题，projectList是否为Seq\[UnaryNode\]，因为Alias之后，单个projection已经具有UnaryNode特性，所以虽然projectList定义为Seq\[NamedExpression\]，但已经可以说是Seq\[UnaryNode\]。在Analyzer中ResolveReferences则对应处理Project返回的UnaryNode的references。所以我们也就理解references作用，其定义为def references: Set\[Attribute\]，也就是将一个UnaryNode的所有需要解析的Attribute都存在references的“集合”里，保证相同的Attribute只需要解析一次。同理output则存储所有的Attribute（Resolved or Unresolved）。

所以，从一个节点（relation）开始，不断添加新的Expression或者LogicalPlan，添加的方式就是Join()，Filter()，Aggregate()，Project()，Distinct()，Sort()，Limit()。如此生成一棵sql树结构。

---

### Analyzer

生成一棵树之后就是要分析这棵树，分析树的结构（Analyzer），分析叶子节点如何处理。所以必然需要一种遍历树的方式（TreeNode提供），需要一些规则来分析节点。这就构成了Analyzer的规则需求，遍历是树结构本身提供的方法，那分析一棵树就是将一些规则添加到遍历的方法中，所以有QueryPlan类的存在，这也是其注释中说明的QueryPlan作用，可以看到默认情况下均采用transformExpressionsDown(rule)。这就解释了小伙伴的博客文章的类图中TreeNode与LogicalPlan之间存在一层QueryPlan的作用。

具体的Analyzer类情况小伙伴博客已经说的比较明白，各种Rule中可以关注下ResolveRelations，ResolveReferences这两种。可以在这个地方设置一个断点，就可以清晰地看到arrayOfString\[0\]是如何进入Rule解析的，也解释了reference值的作用。

    Analyzer.scala - val result = q.resolveChildren(name)

    object ResolveReferences extends Rule[LogicalPlan] {
        def apply(plan: LogicalPlan): LogicalPlan = plan transformUp {
          case q: LogicalPlan if q.childrenResolved =>
            logTrace(s"Attempting to resolve ${q.simpleString}")
            q transformExpressions {
              case u @ UnresolvedAttribute(name) =>
                // Leave unchanged if resolution fails.  Hopefully will be resolved next round.
                val result = q.resolveChildren(name).getOrElse(u)
                logDebug(s"Resolving $u to $result")
                result
            }
        }
      }

另外，从此处也可以跟踪观察遍历的方式与规则，相当于说，这里就是这棵树分析的一处入口。分析是从不同的Rule，也就是不同的角度去解释这棵树的内容，而ResolveReferences是其中必然存在的一个环节。

---

### LogicalPlan

上述过程其实都涉及到一个词，叫做resolve，Analyzer的工作就是将Unresolved转为Resolved，所以回到resolve这个函数实现的LogicalPlan类中。

    options.distinct match {
          case Seq((a, Nil)) => Some(a) // One match, no nested fields, use it.
          // One match, but we also need to extract the requested nested field.
          case Seq((a, nestedFields)) =>
            a.dataType match {
              case StructType(fields) =>
                Some(Alias(nestedFields.foldLeft(a: Expression)(GetField), nestedFields.last)())
              case fields :ArrayType =>
                Some(Alias(nestedFields.foldLeft(a :Expression)(GetArrayField), nestedFields.last)())
              case _ => None // Don't know how to resolve these field references
            }
          case Seq() => None         // No matches.
          case ambiguousReferences =>
            throw new TreeNodeException(
              this, s"Ambiguous references to $name: ${ambiguousReferences.mkString(",")}")
        }

这是Wei Li修改的代码，其中对于包含dot.的情况做细分，查看是否为ArrayType，例如arrayOfStruct.field1.就是需要添加判断arrayOfStruct是否为ArrayType。如果是的，则用GetArrayField解析。此处的局限也是很明显的，再深入一层就跪了，也就是arrayOfStruct.field1.arrayOfStruct.field1，想来这也是比较极端的情况范围了，所以不置考虑。

此处GetArrayField与前面提到的GetArrayOfStructField都是在ComplexTypes.scala文件中添加的case class。这样一层层理解下来也就知道这个case class应该如何构建了，所以仿造GetItem()与GetField()即可。

但是需要注意的是，此处我并没有添加where arrayOfStruct.field1 = true这样的支持，这涉及到Expression的运算比较问题（左边arrayOfStruct.field1这个Expression解析得到的值是ArrayBuffer，右边true这个Expression的值是true，另外Expression有自己的一些比较函数，都可以看看）。SqlPaser.scala文件中包含各种需要解析sql引出的具体方法，可以另外自己跟踪。

---

所以Analyzer过之后，对这棵树进行优化，用到了Optimizer，主要函数流程结构和Analyzer一样，SparkPlan多出一步是为执行做准备，最后执行的时候只要遍历二叉树递归执行就行。因为后续的这些步骤与语义解析关系并不大，所以大家看看小伙伴们的博客就行了（我室友说张包峰会打篮球的，但打得不咋样？奇怪，为什么会想到这茬事情）。自己上面的很多逻辑设计也是半猜半蒙的，所以会有很多不合理的地方（一边看Spark源码，一边学习Scala也是有点痛并快乐着的），望大家看到了提出修改一二。

---

<p style="text-align:right">2014年8月23号</p>

