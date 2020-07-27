---
title: 'Linux技巧系列[1]--awk的妙用'
date: 2019-03-10 10:46:40
categories: Linux系列
---
awk是Linux及Unix环境中现有的功能最强大的数据处理引擎之一，它提供了极其强大的功能：可以进行正则表达式匹配，数学运算符、控制流语句、进程控制语句甚至于内置的变量和函数。可以看到它具备了一个完整的语言所应具有的几乎所有精美特性。<!-- more -->事实上awk就是隐藏在命令行里的编程语言，利用它我们可以很方便的实现数据排序、数据处理、报表生成以及其他许多强大的功能。
#### 1 工作原理
如下图所示，awk脚本由三部分组成，分别对应文本处理的三个步骤。其中BEGIN代码块会在处理文本前执行，中间代码块用于处理pattern表达式匹配到的文本行，而END代码块可以在文本处理完之后进行一些操作。这三个部分都是可选的，无论哪个代码块缺失，剩余的代码块都会按照文本处理流程得到执行。
![1](https://gitee.com/liuyun1995/yun-blog-image/raw/master/Linux%E6%8A%80%E5%B7%A7%E7%B3%BB%E5%88%97%5B1%5D--awk%E7%9A%84%E5%A6%99%E7%94%A8/1.png)

#### 2 基本用法
awk脚本可以直接通过参数传递给awk命令，如下所示，需要指明要处理的文件路径。
```shell
$ awk 'BEGIN{}pattern{commands}END{}' file
```
若脚本比较复杂，也可以将脚本单独存放到一个文件中，然后使用 -f 参数指定脚本文件的位置。
```shell
$ awk -f 脚本文件 file
```
#### 3 内置变量

| 内置变量 | 含义                                            |
| -------- | ----------------------------------------------- |
| $0       | 整行内容                                        |
| \$1-\$n  | 当前行的第1-n个字段                             |
| NF       | 当前行的字段个数，也就是有多少列                |
| NR       | 当前行的行号，从1开始计数                       |
| FNR      | 多文件处理时，每个文件行号单独计数，都是从0开始 |
| FS       | 输入字段分割符，不指定默认以空格或tab键分割     |
| RS       | 输入行分割符，默认回车换行                      |
| OFS      | 输出字段分割符，默认为空格                      |
| ORS      | 输出行分割符，默认为回车换行                    |

假设我们有如下的测试数据：

```shell
$ cat demo1.txt
姓名 性别 年龄 籍贯 学历 职位
张三 男 25 江西 本科 开发工程师
李四 男 34 湖南 博士 算法工程师
李丽 女 26 上海 本科 产品经理
王五 男 29 广东 研究生 开发工程师
```
```shell
$ cat demo2.txt
姓名||性别||年龄||籍贯||学历||职位
张三||男||25||江西||本科||开发工程师
李四||男||34||湖南||博士||算法工程师
李丽||女||26||上海||本科||产品经理
王五||男||29||广东||研究生||开发工程师
```

1）只打印demo1文件的第1，2列的内容。

```shell
$ awk '{print $1,$2}' demo1.txt
姓名 性别
张三 男
李四 男
李丽 女
王五 男
```
说明：因为无需在处理文本前后做一些操作，所以省略BEGIN和END代码块。

2）打印demo1文件每一行有多少列。

```shell
$ awk '{print NF}' demo1.txt
6
6
6
6
6
```

3）打印demo1文件，并在每行前面加上编号。

```shell
$ awk '{print NR,$0}' demo1.txt
1 姓名 性别 年龄 籍贯 学历 职位
2 张三 男 25 江西 本科 开发工程师
3 李四 男 34 湖南 博士 算法工程师
4 李丽 女 26 上海 本科 产品经理
5 王五 男 29 广东 研究生 开发工程师
```

4）只打印demo2文件的第1，3列的内容。

```shell
$ awk 'BEGIN{OFS="||"}{print $1,$2}' demo2.txt
姓名 年龄
张三 25
李四 34
李丽 26
王五 29
```

说明：由于默认的列分割符是空格，因此需要在处理前指定新的列分割符。

#### 4 模式匹配

有时候我们并不需要对每一行都进行处理，而只是需要处理特定的某些行。awk提供了非常强大的模式匹配工具，不仅可以通过正则表达式匹配，还可以通过运算符来匹配，或者通过结合二者来实现更为复杂的匹配。下面列出了可用的运算符列表。

<table>
  <tr><th>符号类型</th><th>符号</th><th>含义</th></tr>
  <tr><td rowspan="8">关系运算符</td><td><</td><td>小于</td></tr>
  <tr><td>></td><td>大于</td></tr>
  <tr><td><= </td><td>小于等于</td></tr>
  <tr><td>>=</td><td>大于等于</td></tr>
  <tr><td>==</td><td>等于</td></tr>
  <tr><td>!=</td><td>不等于</td></tr>
  <tr><td>~</td><td>匹配正则表达式</td></tr>
  <tr><td>!~</td><td>不匹配正则表达式</td></tr>
  <tr><td rowspan="3">逻辑运算符</td><td>&&</td><td>与</td></tr>
  <tr><td>||</td><td>或</td></tr>
  <tr><td>!</td><td>非</td></tr>
</table> 
1）只打印包含李四的行。

```shell
$ awk '/李四/{print $0}' demo1.txt
李四 男 34 湖南 博士 算法工程师
```

2）只打印学历是本科的行。

```shell
$ awk '$5=="本科"{print $0}' demo1.txt
张三 男 25 江西 本科 开发工程师
李丽 女 26 上海 本科 产品经理
```

3）只打印年龄小于30的行。

```shell
$ awk '$3<30{print $0}' demo1.txt
张三 男 25 江西 本科 开发工程师
李丽 女 26 上海 本科 产品经理
王五 男 29 广东 研究生 开发工程师
```
4）只打印年龄小于30并且学历不是本科的行。

```shell
$ awk '$3<30&&$5!="本科"{print $0}' demo1.txt
王五 男 29 广东 研究生 开发工程师
```

#### 5 控制流语句

条件语句格式：

```shell
# 条件语句1
if (条件) {
  ...
}
# 条件语句2
if(条件) {
  ...
} else {
  ...
}
# 条件语句3
if(条件1) {
  ...
} else if(条件2) {
  ...
} else {
  ...
}
```

循环语句格式：

```shell
# for循环示例
for(i = 0; i < 100; i++) {
  ...
}
# while循环示例
while(条件) {
  ...
}
# do-while循环示例
do {
  ...
} while(条件)
```

1）计算下列同学的平均分数，并且只打印平均分大于90分的同学姓名和平均分。

```shell
$ cat demo3.txt
小明 80 90 96 98
李强 93 98 92 91
小红 78 76 87 92
王丽 86 89 68 92
赵斌 85 95 75 90
刘琪 78 88 98 100
$ cat demo.awk
{
  avg = ($2 + $3 + $4 + $5) / 4
  if (avg > 90) {
    print $1,avg
  }
}
$ awk -f demo.awk demo3.txt
小明 91
李强 93.5
刘琪 91
```

2）计算1+2+3+4+...+100的和，分别使用for，while，do-while循环实现。

使用for循环实现：

```shell
$ cat for.awk
BEGIN {
  for(i = 1; i <= 100; i++) {
    sum += i
  }
  print sum
}
$ awk -f for.awk
5050
```

使用while循环实现：

```shell
$ cat while.awk
BEGIN {
  while(i <= 100) {
    sum += i
    i++
  }
  print sum
}
$ awk -f while.awk
5050
```
使用do-while循环实现：

```shell
$ cat doWhile.awk
BEGIN {
  i = 1
  do {
    sum += i
    i++
  } while(i <= 100)
  print sum
}
$ awk -f while.awk
5050
```

#### 6 字符串函数

| 函数名              | 解释                                                  | 函数返回值                |
| ------------------- | ----------------------------------------------------- | ------------------------- |
| length(str)         | 计算字符串长度                                        | 整数长度值                |
| index(str1,str2)    | 在str1中查找str2的位置                                | 返回值为位置索引，从1计数 |
| tolower(str)        | 转换为小写                                            | 转换后的小写字符串        |
| toupper(str)        | 转换为大写                                            | 转换后的大写字符串        |
| substr(str,m,n)     | 从str的m个字符开始，截取n位                           | 截取后的子串              |
| split(str,arr,fs)   | 按fs分割字符串，结果保存到arr数组                     | 切割后的子串的个数        |
| match(str,RE)       | 在str中安装RE查找，返回位置                           | 返回索引位置              |
| sub(RE,RepStr,str)  | 在str中搜索符合RE的子串，将其替换为RepStr，只替换一个 | 替换的个数                |
| gsub(RE,RepStr,str) | 在str中搜索符合RE的子串，将其替换为RepStr，替换所有   | 替换的个数                |

1）搜索字符串 "I have a dream" 中出现 "ea" 字符串的位置。

```shell
$ awk 'BEGIN{str="I have a dream"; location=index(str,"ea"); print location}'
12
```

2）将字符串 "Hadoop is a bigdata Framework" 全部转换为小写。

```shell
$ awk 'BEGIN{str="Hadoop is a bigdata Framework"; print tolower(str)}'
hadoop is a bigdata framework
```
3）搜索字符串 "I got 90 on the test" 第一个数字出现的位置。

```shell
$ awk 'BEGIN{str="I got 90 on the test"; location=match(str,/[0-9]/); print location}'
7
```

4）截取字符串 "I got 90 on the test" 的子串，截取条件是从第3个字符开始，截取6位。

```shell
$ awk 'BEGIN{str="I got 90 on the test"; print substr(str,3,6)}'
got 90
```

5）替换字符串 "I got 90 on the test" 中第一个匹配到的数字串为@符号。

```shell
$ awk 'BEGIN{str="I got 90 on the test"; sub(/[0-9]+/,"@",str); print str}'
I got @ on the test
```

#### 7 综合示例

日常项目中使用到Mybatis作为ORM框架时，我们会遇到根据Java类生成Mysql建表语句的场景，为了减少重复劳动，我们可以通过任意语言来实现这种需求，而使用awk脚本可以用非常少的代码量就能实现此类需求。

1）新建脚本文件create_table.awk，文件内容如下：

```shell
function getUnderScore(str) {
    gsub(/;/, "", str)
    while(match(str, /[A-Z]/)) {
        s1 = substr(str, 0, RSTART - 1)
        s2 = substr(str, RSTART, RLENGTH)
        s3 = substr(str, RSTART + RLENGTH)
        if(s2 == "") { continue }
        if(s1 == "") {
            str = tolower(s2)s3
        } else {
            str = s1"_"tolower(s2)s3
        }
    }
    return str
}

function getMysqlType(javaType) {
    if(javaType ~ /String/) {
        return "VARCHAR(255)"
    }
    if(javaType ~ /int|Integer/) {
        return "INT"
    }
    if(javaType ~ /long|Long/) {
        return "BIGINT"
    }
    if(javaType ~ /float|Float/) {
        return "FLOAT"
    }
    if(javaType ~ /double|Double/) {
        return "DOUBLE"
    }
    if(javaType ~ /boolean|Boolean/) {
        return "BIT"
    }
    if(javaType ~ /BigDecimal/) {
        return "DECIMAL"
    }
    if(javaType ~ /Date/) {
        return "DATE"
    }
    return "NULL"
}

$0~/[a-zA-Z]([a-zA-Z0-9])*/{
    if(match($0, /.*class.*/)) {
        tableName = getUnderScore($3)
        printf("DROP TABLE `%s`;\n", tableName)
        printf("CREATE TABLE `%s` (\n", tableName)
    } else {
        columnName = getUnderScore($3)
        columnType = getMysqlType($2)
        if(key == "") { key = columnName }
        printf("    `%s` %s NOT NULL,\n", columnName, columnType)
    }
}

END{
    printf("    PRIMARY KEY (`%s`)\n", key)
    printf(") ENGINE=InnoDB DEFAULT CHARSET=utf8;\n")
}
```

2）新建测试文件Product.java，文件内容如下：

```java
public class Product {
    private Long id;
    private String productName;
    private BigDecimal productPrice;
    private Boolean isAvailable;
    private Date createTime;
}
```

3）执行create_table.awk脚本，输出结果如下：

```shell
$ awk -f create_table.awk Product.java
DROP TABLE `product`;
CREATE TABLE `product` (
    `id` BIGINT NOT NULL,
    `product_name` VARCHAR(255) NOT NULL,
    `product_price` DECIMAL NOT NULL,
    `is_available` BIT NOT NULL,
    `create_time` DATE NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

#### 8 总结

本文我们详细介绍了awk这门工具的基本使用，了解了awk的工作原理，知道了它所擅长的一些领域。其实描述awk为Linux环境下的命令行工具并不准确，它其实是一门编程语言，它不仅有内置的变量和函数，还具有流程控制语句，甚至我们可以自定义变量和函数。它的语法参照了C语言的编程风格，而且具有强大的动态类型推导能力，因此它比shell脚本更易于编程。确实，在很多方面可以用awk脚本来替代shell脚本帮助我们完成一些自动化的事情，因此熟练掌握好awk可以有效的提升我们的工作效率。