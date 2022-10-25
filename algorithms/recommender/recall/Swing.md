用户 u 和用户 v，都购买过同一件商品i ，则三者之间会构成一个类似秋千的关系图。若用户 u 和用户 v 之间除了共同购买过 i 外，还共同购买过商品 j，则认为两件商品i和j是具有某种程度上的相似的。
也就是说，商品与商品之间的相似关系，是通过用户关系来传递的。为了衡量物品 i 和 j 的相似性，考察都购买过物品 i 和 j 的用户 u 和用户 v ， 如果这两个用户共同购买的物品越少，则物品 i 和 j 的相似性越高。
相似度公式计算公式如下：

$s(i, j) = \sum_{u\in(u_i\cap u_j)} \sum_{v\in(u_i\cap u_j)} \frac{1}{\alpha + \| i_u \cap j_u \| }$

其中：
 - $u_i表示购买了i的用户集合$
 - $u_j表示购买了j的用户集合$
 - $i_u表示u购买的物品集合$
 - $i_v表示v购买的物品集合$

公式比较简单，先从数据集分别构建商品-用户map和用户-商品map并broadcast出去，然后商品之间笛卡尔积构建商品对，商品对之间应用上述公式计算相似度score