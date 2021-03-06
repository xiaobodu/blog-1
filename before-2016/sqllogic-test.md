# 使用sqllogic test进行SQL逻辑测试

[sqllogictest](http://sqlite.org/sqllogictest/doc/trunk/about.wiki)是sqlite的一套sql logic 测试集合，它不光支持对sqlite的测试，还支持不同database（包括MySQL），所以可以说是开发数据库进行逻辑测试的利器。

sqllogic test的脚本格式异常简单，看文档一下子就明白怎么弄了，这里简单记录一下。

sqllogic test的每一个test case之间，都是使用空行分隔，case主要有两种，statement和query。

## statement 

statement对应的其实就是非select操作，譬如insert，delete，update等，statement case最开始是如下两行中的一行：

```
statement ok
statement error
```

ok表明这个statement执行要成功，如果失败了，就返回错误，而error则是相反，如果语句执行成功，就是case失败。statement的例子很好理解，譬如：

``` statement ok
CREATE TABLE tab0(pk INTEGER PRIMARY KEY, col0 INTEGER, col1 FLOAT, col2 TEXT, col3 INTEGER, col4 FLOAT, col5 TEXT)
```

这个case就是创建一个tab0的表，一定要成功。

## query

query对应的就是select，整天被SQL折磨的童鞋应该知道，select应该是最复杂也是最重要的一条语句，我们多数的业务其实也就是在考虑如何用select来进行数据处理。sqllogic test里面的test case最多的也是select。而它的格式也是最复杂的。

query首先以如下格式开头：

```
query <type-string> <sort-mode> <label>
```

`<type-string>` 用来表明这次查询会返回多少列，每个列的类型，我们使用I表明interger，R表明float，以及T表明text，为毛只有这么几种基本格式，主要在于sqllogic test是用来测试SQL logic正确的，不是用来测试SQL数据是不是移除，非法这些的。如果type string是III，表明返回的结果是三列的，每列都是integer，这里我们需要注意，即使type string是I，但是我们可能得到一个float，或者text，这时候就要做类型转换了。这个坑我是在跑test case的时候发现的，竟然返回的数据类型跟指定的不一致，指定的是I，但返回的竟然是"abc"这种无法转成integer的字符串，然后只能全当0来处理。

sqllogic test在用type string得到对应的数据之后，会将其全部转成string供后续处理，如果是NULL，返回"NULL"，对于integer，就是用`%d`进行format，而对于float，则是`%.3f`，对于text，如果是空，则是"(empty)"，如果含有控制字符或者无法打印的字符，则用`@`替换。

`<sort-mode>`是一个可选项，用来指定如何处理得到的result set。它可能有"nosort"（完全不排序），"rowsort"（对结果集按照行进行排序），以及"valuesort"（对结果集里面的所有数据全部排序）。默认是"nosort"。sort采用的是字节排序的方式，也就是是"9"在”10“的后面。

`<label>`主要是用来判断相同label的query执行是不是有相同的hash结果，这个主要是用来测试不同的SQL语句，譬如`select * from t where a && b`和`select * from t where a and b`这种的结果是不是一致的。因为这些语句虽然不一样，但是在内部做了很多转换之后会得到一致的execute plan。

紧跟在query之后，就是select语句了，譬如

```
SELECT pk FROM tab0 WHERE col3 < 6 OR col3 < 1 AND col3 BETWEEN 2 AND 7
----
```

select语句跟下面的结果使用`----`进行分割。

对于结果集，如果为空，则什么也没有，如果有结果集，则又有两种情况。

首先就是完全将结果列出来的，譬如这种的

```
SELECT pk FROM tab0 WHERE col0 > 283 AND col1 > 969.7
----
54
74
76
92
```

这里select会得到4行数据，我们需要在自己的测试程序里面显示的跟这里的数据进行匹配，看是否一致。

但有时候，一些select可能返回特别多的数据，如果全列出来，对于test case显得过于庞大，所以这时候就有另一种格式的，如下：

```
SELECT pk FROM tab0 WHERE col3 <= 682
----
66 values hashing to 0269a17d465888e8818debfeede53627
```

上面的select会返回66条数据，这些数据的md5 hash为0269a17d465888e8818debfeede53627。对于我们的测试程序来说，这时候就需要判断返回的数据是不是66，以及计算的hash是不是跟上面相等了。

# 控制语句

sqllogic test还支持一些控制语句，譬如 halt，这个主要是用来调试用的，如果我们跑一个case出错了，可以在后面加上halt，这样下面的case就不会跑了。

还有`hash-threshold <max-result-set-size>`， 这个语句主要是用在sqllogic test的case生成上面，因为我们只是用sqllogic test的case进行测试，所以没有关心。

# 条件语句

因为sqllogic test是一个通用的测试集合了，但是有些语句是一些数据库特有的，为了保证使用不同的database都能过这套测试集合，sqlloigc test提供了skipif以及onlyif这两个控制语句。

譬如，如果skipif mysql，则表明如果使用MySQL，则不跑紧跟着的下个case，如果是onlyif mysql，则表明只有MySQL才能运行下面这个测试。

