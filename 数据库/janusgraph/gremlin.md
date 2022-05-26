# gremlin

### 1. Gremliin总结

g.V()查询顶点

g.E()查询边

e.E().id()查询边的id

g.V().label() 顶点的label

g.E().label() 边的label

g.V().properties()顶点的属性，List形式

g.V().properties().key() 顶点的属性名称

g.V().valueMap() 所有顶点的属性，结果以array数组形式

g.V().values() 顶点的属性值 ==> _g.V().properties().value()_

g.V().out() 访问顶点的out方向邻接点

g.V('3:TinkerPop').out() 访问某个顶点的out方向邻接点

g.V('3:TinkerPop').out('define') 限制仅“define”类型的边相连的顶点

g.V('3:TinkerPop').in() 访问某个顶点的IN方向邻接点

g.V('3:TinkerPop').in('implements') 限制了关联边的类型

g.V('3:TinkerPop').both() 访问某个顶点的双向邻接点

g.V('3:TinkerPop').outE() 访问某个顶点的OUT方向邻接边

g.V('3:TinkerPop').inE() 访问某个顶点的IN方向邻接边

g.V('3:TinkerPop').inE('implements') 且限制了关联边的类型

g.V('3:TinkerPop').bothE() 访问某个顶点的双向邻接边

g.V('3:TinkerPop').inE().outV() 访问某个顶点的IN邻接边 的 出顶点

g.V().count() 查询图中所有顶点的个数

* `count()`: 统计查询结果集中元素的个数；
* `range(m, n)`: 指定下界和上界的截取，左闭右开。比如`range(2, 5)`能获取第2个到第4个元素（0作为首个元素，上界为-1时表示剩余全部）；
* `limit(n)`: 下界固定为0，指定上界的截取，等效于`range(0, n)`，语义是“获取前`n`个元素”。比如`limit(3)`能获取前3个元素；
* `tail(n)`: 上界固定为-1，指定下界的截取，等效于`range(count - n, -1)`，语义是“获取后`n`个元素”。比如`tail(2)`能获取最后的2个元素；
*   `skip(n)`: 上界固定为-1，指定下界的截取，等效于`range(n, -1)`，语义是“跳过前`n`个元素，获取剩余的元素”。比如`skip(6)`能跳过前6个元素，获取最后2个元素。

    **`path()`，获取当前遍历过的所有路径**

    **`simplePath()`，过滤掉路径中含有环路的对象，只保留路径中不含有环路的对象**

    **`cyclicPath()`，过滤掉路径中不含有环路的对象，只保留路径中含有环路的对象**

    * `repeat()`: 指定要重复执行的语句，如`repeat(out('friend'))`
    * `times()`: 指定要重复执行的次数，如执行3次`repeat(out('friend')).times(3)`
    * `until()`: 指定循环终止的条件，如一直找到某个名字的朋友为止`repeat(out('friend')).until(has('name','xiaofang'))`
    * `emit()`: 指定循环语句的执行过程中收集数据的条件，每一步的结果只要符合条件则被收集，不指定条件时收集所有结果
    *   `loops()`: 当前循环的次数，可用于控制最大循环次数等，如最多执行3次`repeat(out('friend')).until(loops().is(3))`

        g.V('okram') .repeat(out()) .until(has('name', 'Gremlin')) .emit(hasLabel('person')) .path()

`emit()`与`until()`搭配使用时，是“或”的关系而不是“与”的关系，满足两者间任意一个即可

查找两点之间的最短路径

g.V('okram') .repeat(bothE().otherV().simplePath()) .until(hasId('javeme').and().loops().is(lte(3))) .hasId('javeme') .path()

遍历器中的元素是属性

order().by(incr)

遍历器中的元素是顶点或边时

order().by(key, incr)

* `group()`: 对结果集进行分组，可通过by(property)来指定根据什么维度进行分组，可称维度为分组键；如果不指定维度则以元素id作为分组键，相当于重复的元素被分为一组。每一组由分组键+组内元素列表构成。如果有需要也可对每一组的元素列表进行reduce操作，依然使用by()语句，如by(count())对组内元素计数。
* `groupCount()`: 对结果集进行分组，并统计每一组的元素个数。每一组由分组键+组内元素数量构成。
* `dedup()`: 去除结果集中相同的元素，可通过`by(property)`来指定根据什么维度进行去重。
* `by()`: 语义上一般指“根据什么维度”，与上述语句配合使用，如`group().by()`、`dedup().by()`等。也可与其它语句配合，如前面讲到的排序`order().by()`及路径`path().by()`等。\*\*\*\*
