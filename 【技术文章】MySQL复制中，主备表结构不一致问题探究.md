### 【技术文章】MySQL复制中，主备表结构不一致问题探究
最近有研发同学的大表想修改数据decimal精度，想先做备库DDL，备库完成后切换主从再重复同样动作。数据库操作从decimal(23,6)提升精度到decimal(27,8)在MySQL中是要锁表，研发担心影响在线业务(24小时业务)。

alter table tab_x modify  xnum decimal(27,8) not null default 0.0;
对于每次的手工操作都习惯先测试预演一次，以排除部分理所应当、常识性的错误。在测试过程中发现不能消费relay log，其中错误日志是：
```
Source：
CREATE TABLE `t1` (
  `c1` int(11) DEFAULT NULL,
  `c2` int(11) DEFAULT NULL,
  `c3` int(11) DEFAULT NULL,
  `xnum` decimal(23,6) NOT NULL DEFAULT '0.000000'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
Replica ：
CREATE TABLE `t1` (
  `c1` int(11) DEFAULT NULL,
  `c2` int(11) DEFAULT NULL,
  `c3` int(11) DEFAULT NULL,
  `xnum` decimal(27,8) NOT NULL DEFAULT '0.00000000'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

同步链路正常情况下执行SQL
insert into t1(c1,c2,c3,xnum)  values(10,12,11,1.990001);
在备库你会收到错误如下
Last_SQL_Errno: 1677
Last_SQL_Error: Column 3 of table 'test.t1' cannot be converted from type 'decimal(23,6)' to type 'decimal(27,8)'
```

看到报错后没有仔细深究解决办法就给同事反馈操作可能不支持，但是在自己印象中好像字符串加长是能支持。直到这次又有一个同事对varchar加长想试试备库操作能成功不，再测试发现问题依旧。跟印象出偏差的时候就要去找到知识点，不断完善自己的知识体系。
后续自己空闲下来去阅读官方文档，其实这些都能支持，发现向同事输出了不完全的信息及时找相关研发同学沟通对齐认识偏差。

以上为文章写作背景。
(因为黑命贵的原因，以下 源【Source】等同于主库 ，副本【Replica】等同于备库、从库 )

### 用于复制的源表和目标表不必相同
主库上的一个表的列可以多于或少于该表的备库本的列。此外，根据某些条件，主从上的相应表列可以使用不同的数据类型。
下面我们来具体说说主从之间哪些不同的表结构定义能被主从复制协议支持。

在主库或备库上具有更多列的复制
您可以将表从主库复制到备库，并且表的主库和备库具有不同的列数，但要满足以下条件:

1、表的两个版本的公共列必须在主从以相同的顺序定义
    (即使两个表的列数相同，也是如此)

2、表的两个版本的公共列必须在任何附加列之前定义
    这意味着在从库上执行 ALTER TABLE 语句时，如果在两个表共有的列范围内向表中插入一个新列，将导致复制失败，如下面的示例所示：

 ```
假设主备上的表 t 由下面的 CREATE TABLE 语句定义:
CREATE TABLE t (c1 INT,c2 INT,c3 INT);
假设这里显示的 ALTER TABLE 语句在备库上执行是不会中断复制本的。
ALTER TABLE t ADD COLUMN cnew1 INT AFTER c3;
在副本上允许使用前一个 ALTER TABLE，
因为表 t 的两个版本共有的列 c1、 c2和 c3在表的两个版本中都保留一起，在任何不同的列之前。
但是执行ALTER TABLE t ADD COLUMN cnew2 INT AFTER c2;
会导致主备复制中断，因为新的列 cnew2位于两个公共列之间
```
3、具有更多列的表(无论主备)中的每个“额外”列都必须有一个默认值

    此外，当表的从库的列多于主库的列时，表的每个公共列必须在两个表中使用相同的数据类型。

下面的例子说明了一些有效和无效的表定义：



### 主库上有更多列

下表定义有效并且正确复制：
```
source> CREATE TABLE t1 (c1 INT, c2 INT, c3 INT);
replica>  CREATE TABLE t1 (c1 INT, c2 INT);
```
下面的表定义将引发一个错误，因为主备的表的公共列的定义在备库上的顺序与在主库上的顺序不同:
```
source> CREATE TABLE t1 (c1 INT, c2 INT, c3 INT);
replica>  CREATE TABLE t1 (c2 INT, c1 INT);
```
下面的表定义也会引发一个错误，因为主库上的额外列的定义出现在主从共同列的定义之前:
```
source> CREATE TABLE t1 (c3 INT, c1 INT, c2 INT);
replica>  CREATE TABLE t1 (c1 INT, c2 INT);
```
### 从库上有更多列

下列表定义有效并正确复制:
```
source> CREATE TABLE t1 (c1 INT, c2 INT);
replica>  CREATE TABLE t1 (c1 INT, c2 INT, c3 INT);
```
下面的定义引发了一个错误，因为主备的公共列在主库和备库上的定义顺序不同:
```
source> CREATE TABLE t1 (c1 INT, c2 INT);
replica>  CREATE TABLE t1 (c2 INT, c1 INT, c3 INT);
```
下表定义也引发了一个错误，因为表的备库的额外列的定义出现主备的公共列的定义之前:
```
source> CREATE TABLE t1 (c1 INT, c2 INT);
replica>  CREATE TABLE t1 (c3 INT, c1 INT, c2 INT);
```
下面的表定义失败，因为备库与主库相比有额外的列，而且主备的公共列 c2使用不同的数据类型:
```
source> CREATE TABLE t1 (c1 INT, c2 BIGINT);
replica>  CREATE TABLE t1 (c1 INT, c2 INT, c3 INT);
```

### 复制具有不同数据类型的列
理想情况下，同一个表的主从库上的对应列应该具有相同的数据类型。然而，只要满足某些条件，这并不总是得到严格执行。

通常可以从一个给定数据类型的列复制到另一个具有相同类型、相同大小或宽度(如果适用)或更大的列。例如，您可以从 CHAR (10)列复制到另一个 CHAR (10)列，或者从 CHAR (10)列复制到 CHAR (25)列，没有任何问题。在某些情况下，还可以从具有一种数据类型的列(在主库上)复制到具有不同数据类型的列(在备库上) ; 当源文件版本的数据类型被提升为与副本大小相同或更大的类型时，这被称为属性提升。

基于语句的复制。当使用基于语句的复制时，一个简单的经验法则是，“如果在源上运行的语句也能在副本上成功执行，那么它也应该能够成功复制”。换句话说，如果语句使用的值与副本上给定列的类型兼容，则可以复制该语句。例如，您可以将适合 TINYINT 列的任何值插入到 BIGINT 列中; 因此，即使您将表的副本中 TINYINT 列的类型更改为 BIGINT，对源的该列的任何成功插入也应该在副本上成功，因为不可能有一个大到足以超过 BIGINT 列的合法 TINYINT 值。

基于行的复制: 属性升级和降级。基于行的复制支持在较小的数据类型和较大的类型之间进行属性升级和降级。还可以指定是否允许降级的列值进行有损(截断)或无损转换。

有损和无损转换。如果目标类型不能表示要插入的值，则必须决定如何处理转换。如果允许转换，但截断(或以其他方式修改)源值以实现目标列中的“匹配”，则进行所谓的损耗转换。不需要截断或类似的修改来适应目标列中的源列值的转换是无损耗转换。

从 slave_type_conversions全局服务器变量的设置控制在副本上使用的类型转换模式。该变量从下表中获取一组值，该值显示了每种模式对副本类型转换行为的影响:

Mode 模式	Effect 效果
1. ALL_LOSSY

    在这种模式下，允许类型转换，这意味着信息丢失。
这并不意味着允许无损转换，只是允许只有需要有损转换或根本不需要转换的情况; 例如，只允许此模式允许将 INT 列转换为 TINYINT (有损转换) ，而不允许将 TINYINT 列转换为 INT 列(无损转换)。在这种情况下，试图进行后一种转换将导致复制停止，并在副本上出现错误。

2. ALL_NON_LOSSY	

   此模式允许不需要截断源值或对源值进行其他特殊处理的转换; 也就是说，它允许目标类型的范围比源类型的范围更广的地方进行转换。
   设置此模式与是否允许损耗转换没有关系; 这是由 ALL_lossy 模式控制的。如果只设置了ALL_NON_LOSSY，而没有设置 ALL_LOSSY，那么试图进行一个会导致数据丢失的转换(比如 INT 到 TINYINT，或者 CHAR (25)到 VARCHAR (20)) ，则会导致副本因错误而停止。

3. ALL_LOSSY,ALL_NON_LOSSY	

   设置此模式时，允许所有受支持的类型转换，无论它们是否为有损转换。

4. ALL_SIGNED
    
   将提升的整数类型视为有符号值(默认行为)。

5. ALL_UNSIGNED	

    将提升的整数类型视为无符号值。

6. ALL_SIGNED,ALL_UNSIGNED	

    如果可能，将提升的整数类型视为有符号类型，否则视为无符号类型。

7. [empty 空的]	

    如果备库没有设置slave_type_conversions，则不允许进行属性升级或降级; 这意味着源表和目标表中的所有列必须属于相同的类型。这个模式是默认的。



支持不同但类似的数据类型之间的转换如下表所示:
```
- 在任何整数类型 TINYINT、 SMALLINT、 MEDIUMINT、 INT 和 BIGINT 之间
- 在任何小数类型 DECIMAL、 FLOAT、 DOUBLE 和 NUMERIC 之间。
- 在任何字符串类型 CHAR、 VARCHAR 和 TEXT 之间，包括不同宽度之间的转换。
- 任何二进制数据类型之间的 BINARY、 VARBINARY 和 BLOB，包括不同宽度之间的转换。
- 在任意2个大小的2 BIT 列之间。
除了上述5类的类型转换都是不被允许的，否则会导致主从复制中断。
```
一般我们的操作都是无损的(ALL_NON_LOSSY)，只需要在备库设置slave_type_conversions重启复制进程就可以支持到主备结构不一致的复制可以正常同步。