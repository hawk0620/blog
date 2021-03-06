# 理解数据库索引

在日常开发中，数据库降低了我们操作数据的门槛，我们只要编写特定的 SQL，就能对数据做增删改查操作。在简化的背后，往往都隐藏着性能优化的福利，数据库也是如此，我们知道假如没有索引，查询数据就会全表扫描，而索引就如书的目录一般，大大提高了查询效率。本文将对数据库索引进行介绍，认识索引的数据结构，同时也介绍索引的其他概念。

## 索引的数据结构

索引在本质上是为了优化查找的速度，对于给定的数据，我们可以使用顺序查找，如果数据已经排好序，我们可以使用二分查找，如果查找的数据量不大，我们可以构造二叉查找树将查找放在内存中，而索引的数据结构是由平衡二叉树演化而来，在正式介绍索引的数据结构之前，让我们先来看看二叉查找树。

### 二叉查找树

二叉查找树要求左子树的键值总是小于根的键值，右子树的键值总是大于根的键值。

![1527218750414](https://user-images.githubusercontent.com/5633917/40524676-690b3df0-600e-11e8-9b95-fc14ca3874d0.jpg)

![1527217930423](https://user-images.githubusercontent.com/5633917/40524320-918f2bf8-600c-11e8-8c84-a2b44ae5da8b.jpg)

二叉查找树的问题是假如单支过长就会大大影响其查找效率，甚至退化成顺序查找

![1527218408243](https://user-images.githubusercontent.com/5633917/40524536-994d0d46-600d-11e8-9b6b-0bc62971d58c.jpg)

为了提高二叉树的查找效率，需要构造的这棵二叉树是平衡的——平衡二叉树要求任何结点的两个子树的高度最大差为1。平衡二叉树通常需要左旋、右旋来达到平衡。

![1527219158761](https://user-images.githubusercontent.com/5633917/40524836-711c70ee-600f-11e8-957e-e55fc7ce9ec0.jpg)


### B* 树

#### B- 树

我们先来看看对 B 树的描述：

- B 树的 B 不是表示二叉，而是表示平衡
- B 树并不是一颗二叉树，B 树是 n 叉的（n > 2）
- 每个结点有多个关键字，关键字之间又有指向孩子结点的指针
- 一个结点内的关键字都有序排列
- 所有叶子结点都在同一层

![1527237093708](https://user-images.githubusercontent.com/5633917/40534616-2eb9a6ca-6039-11e8-9825-0b2a368046d9.jpg)

对于查找而言，B 树的查找类似二叉树，因为每个结点内的关键字都是排序好的 key[1...n]，我们可运用二分查找将查找关键字 k 与 key[i] 比较，从而找出相应区间的子树。

B 树查找的简化代码：

```c
Result BTreeSearch(BTNode *t, KeyType k) {
    BTNode *p = t; *q = NULL; // q 指向 p 的双亲
    int found = 0, i = 0;
    while (p != NULL && found != 0) {
        i = BinarySearch(p, k);
        if (i > 0 && p->key[i] == k) {
            found = 1;
        } else {
            q = p;
            p = p->ptr[i];
        }
    }
    ...
}
```

上述代码也可以使用递归。有了这些基本的认识后，不难发现 B 树的查找效率与树的高度有关，高度越小，查找的次数就越少。

接下来看看 B 树的插入和删除。
对插入而言，如果该结点还有空位置，直接插入，否则，会将结点分成两部分，中间位置的关键字插入到父结点中，如果父结点也不满足，再往上插，直到这个过程传到根结点。

![1527237417928](https://user-images.githubusercontent.com/5633917/40534843-e6d25d24-6039-11e8-8ad3-7dc8825f3a51.jpg)

插入15

删除比插入稍微复杂一点，如果删除一个关键字后，结点的关键字个数没有少于它的装填因子，则直接删除

![1527237872376](https://user-images.githubusercontent.com/5633917/40535168-effe9d9e-603a-11e8-9227-967f49c13d56.jpg)

删除8，16

否则分两种情况：

- 如果左（右）兄弟结点的关键字个数大于装填因子，则将左（右）结点最大或最小关键字上移到父结点，再把父结点中大于或小于上移关键字的关键字下移到要被删除的结点中。

![1527582084766](https://user-images.githubusercontent.com/5633917/40646654-663aaaf6-635c-11e8-8ec9-af89bc8727cd.jpg)

删除15

- 如果左（右）兄弟结点的关键字个数等于装填因子，这时需要把删除关键字的结点与左（右）结点关键字和分割二者关键字的双亲结点关键字合并成一个结点，如果因此使双亲结点关键字个数小于装填因子，则对双亲结点也做同样处理，以致可能使整棵树的高度减少一层。

![1527238316921](https://user-images.githubusercontent.com/5633917/40535534-f8a9475e-603b-11e8-8577-54b1eccfe6a4.jpg)

删除4后的结果

由上可知，B 树的插入和删除都是需要代价的，所以我们对数据库索引的建立也需要特别谨慎，否则不合理的索引反而降低了效率。

#### B+ 树

B+ 树是 B- 树的变形，常用于索引结构中，它与 B- 树的主要差异有：

- B+ 树中所有叶子结点包含了全部关键字，即非叶子结点的关键字也出现在叶子结点中
- 叶子结点的指针不再指向另一级索引，而是直接指向数据文件的记录
- 分支结点不包含关键字对应的存储地址，只包含指向各个子结点的指针
- 所有叶子结点链接成一个线性链表

![1527582980853](https://user-images.githubusercontent.com/5633917/40647474-84e7313e-635e-11e8-9353-4580813a539b.jpg)

B 树和平衡二叉树的一个重要区别是结点的大小及其造成的树的高度不同，B+ 树的结点大小一般是一个磁盘块的大小，也就是数据页的大小，因此 B 树矮而胖，二叉树高而瘦。前面已经提到 B 树的查找效率和其高度有关，假设当前数据表的数据为 N，每个磁盘块的数据项的数量是 m，则有 h = ㏒(m+1)N，而 m = 磁盘块的大小 / 数据项的大小，磁盘块的大小又是固定的，故数据项的大小越小，树的高度也就越小。这就是为什么要求索引字段尽可能小的原因。同理，将数据不存储在分支结点，也是为了尽可能多的存放数据项。

## B+ 树索引

B+ 树索引就是 B+ 树在数据库中的实现。B+ 索引在数据库中有一个特点是高扇出性，因此在数据库中，B+ 树的高度一般都在2～4层，这也就是说查找某一键值的行记录时最多只需要2到4次 IO。

B+ 树索引并不能找到一个给定键值的具体行。B+树索引能找到的只是被查找数据行所在的页。然后数据库通过把页读入到内存，再在内存中进行查找，最后得到要查找的数据。

### 聚集索引和辅助索引

B+ 树索引可分为聚集索引和辅助索引（secondary index），它们的主要区别是聚集索引要求以唯一的 key（一般是主键）来构造索引，文件中记录的物理存储顺序和索引顺序一致，由于实际的数据页只能按照一棵 B+ 树进行排序，因此每张表只能拥有一个聚集索引，辅助索引的 key 可以不是唯一的，辅助索引能提高聚集索引以外 key 的查找性能，这也会增加一定的开销。

下面表有三个列，分别是 id（主键）、name 和 salary，我们来看看聚集索引和辅助索引的原理：

![1527560106240](https://user-images.githubusercontent.com/5633917/40635134-245f48c0-632b-11e8-9ff7-40612778be09.jpg)

以上是聚集索引

![1527560883816](https://user-images.githubusercontent.com/5633917/40635179-5a58fb60-632b-11e8-8b25-090e06820808.jpg)

以上是辅助索引

### 联合索引

我们已经介绍过了在单个列上使用索引，联合索引是指对表上的多个列进行索引，联合索引的本质也是一棵 B+ 树，下面看看联合索引的内部结构：

![dff64b61-998e-47be-910d-02b596227f71](https://user-images.githubusercontent.com/5633917/40536633-6a475164-603f-11e8-830a-24bd0b69d702.png)

对上图而言，（1，1）、（1，2）、（2，1）、（2，4）、（3，1）、（3，2），数据按（a，b）的顺序进行了存放，第一列是升序排序的，第二列是根据第一列排序而排序的。

因此，对于查询 `SELECT*FROM TABLE WHERE a=xxx and b=xxx`，显然是可以使用（a，b）这个联合索引的。对于单个的a列查询 `SELECT*FROM TABLE WHERE a=xxx`，也可以使用这个（a，b）索引。但对于b列的查询 `SELECT*FROM TABLE WHERE b=xxx`，则不可以使用这棵B+树索引。可以发现叶子节点上的b值为1、2、1、4、1、2，显然不是排序的，因此对于b列的查询使用不到（a，b）的索引。

联合索引能在索引到第一个键值后对第二个键值进行排序。例如，查询某个用户的购物情况，并按照时间进行排序，最后取出最近三次的购买记录，这时使用联合索引可以避免多一次的排序操作，因为索引到某个用户 id 后，购买记录已经是有序的了。

正如前面所介绍的那样，联合索引（a，b）其实是根据列a、b进行排序，因此语句 `SELECT...FROM TABLE WHERE a=xxx ORDER BY b` 可以直接使用联合索引得到结果。

然而对于联合索引（a，b，c）来说，语句 `SELECT...FROM TABLE WHERE a=xxx AND b=xxx ORDER BY c` 或 `SELECT...FROM TABLE WHERE a=xxx ORDER BY b` 同样可以直接通过联合索引得到结果。

但对于语句 `SELECT...FROM TABLE WHERE a=xxx ORDER BY c `，联合索引不能直接得到结果，因为 c 是用不到索引的。

这就是索引最左前缀匹配的特性。根据该原则，我们建立联合索引时要考虑好查询尽可能地用得上索引，这也要求我们尽可能选择区分度高的列作为索引。

### 小结

- 在作为主键的列上，强制该列的唯一性和组织表中数据的排列结构。
- 在经常用在连接的列上，这些列主要是一些外键，可以加快连接的速度。
- 在经常需要根据范围进行搜索的列上创建索引，因为索引已经排序，其指定的范围是连续的。
- 在经常需要排序的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间。
- 在经常使用在where子句中的列上面创建索引，加快条件的判断速度。

## 哈希索引和位图索引

最后再简单介绍下另外两种索引结构

### 哈希索引

哈希索引通过哈希算法来实现查找，其冲突解决采用链地址法，我们知道哈希算法的时间复杂度为 O(1)，所以哈希索引是非常高效的。

因为哈希索引的记录不以任何特定方式排序，这也导致哈希索引无法应用在范围查找中。

### 位图索引

位图索引（bitmap index）是为多个列查询设计的特殊索引，位图索引适合用于列上的值大量重复出现。

表结构：

ID | gender | income_level
--- | --- | ---
43123 | m | L1
65654 | f | L2
76534 | f | L1
12343 | m | L4
65765 | f | L3

gender 的位图：

|  |  |
| --- | --- |
| m | 10010 |
| f | 01101 |

income_level 的位图：

|  |  |
| --- | --- |
| L1 | 10100 |
| L2 | 01000 |
| L3 | 00001 |
| L4 | 00010 |
| L5 | 00000 |

上述表对于只以性别为条件的查询，位图索引并不能带来什么性能的提升。然而对查询 `Select * from t where gender = 'f' and income_level = 'L3'`，位图索引会执行两个位图的交操作（逻辑与）。即 gender 的位图 = f(01101) 和 income_level 的位图 = L2(01000) 的交得到位图 01000。显然对于多个列上大量重复数据项的查询，位图索引可以提高查找效率。此外，位图索引还有体积小的优点。

## 总结

本文是我学习数据库索引的笔记，仅仅介绍了数据库的几种索引的原理，并没有深入到更加底层的研究，只能对日常开发中如何建立索引、选择索引起到一定的指示作用，而对于查询性能的优化还是需要从大量的实践中总结出经验。

#### 参考文章

[MySQL索引原理及慢查询优化](https://tech.meituan.com/mysql-index.html)

[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)

