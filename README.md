
MongoDB是典型的非关系型数据库，但是它的功能越来越复杂，很多项目中，我们为了快速拓展，甚至直接使用Mongo 来替代传统DB做数据持久化。虽然MongoDB在支持具体业务时没有问题，但是由于它是文档型数据库，拥有一套独立的语法，不再支持传统的SQL。开发人员发现在实际开发过程中，由于语法问题，在处理复杂的业务查询时，不知该如何下手，使不上劲。在这里我总结了一下接触到的使用场景：如果是简单的业务，那么我们直接使用spring JPA来实现就可以，比如这些操作：1、创建2、删除3、修改4、简单的查询因为这些语句的逻辑往往不是很复杂，JPA完全可以胜任，而且还清晰直观。如果是复杂的场景，我们就使用MongoTemplate 来组织条件逻辑：假设背景是有张Student 表，结构如下：我们已经预先插入了下边的数据：


![](https://img2024.cnblogs.com/blog/704073/202412/704073-20241203143754028-266698577.jpg)


（1）先来一个简单的单条件查询：




```
1     private void simpleInQuery() {
2         Query query = new Query();
3         query.addCriteria(Criteria.where("classNo").ne(2));
4         List students = mongoTemplate.find(query, Student.class);
5         log.warn("the query result is: {}", students);
6     }
```


输出如下：




```
14:39:19.805  WARN 83348 --- [           main] c.e.demo.learn.mongo.MongodbController   : the query result is: 
[Student(_id=674d8125bf8e9e35bbc08718, name=xiaoa, description=good1, classNo=1, age=15), 
Student(_id=674d8125bf8e9e35bbc08719, name=xiaob, description=good2, classNo=1, age=13), 
Student(_id=674d8125bf8e9e35bbc0871a, name=xiaoc, description=good3, classNo=1, age=15),Student(_id=674d8125bf8e9e35bbc0871b, name=xiaod, description=good4, classNo=1, age=15), 
Student(_id=674d8153c993425aaa5c4fec, name=zhongd, description=perfect2, classNo=3, age=15), 
Student(_id=674d81dc8130705614f23311, name=biga, description=nice, classNo=3, age=15), 
Student(_id=674d81dd8130705614f23312, name=bigb, description=nice, classNo=3, age=13), 
Student(_id=674d81dd8130705614f23313, name=bigc, description=nice, classNo=3, age=15), 
Student(_id=674d81dd8130705614f23314, name=bigd, description=nice, classNo=3, age=15)]
```


观察代码，我们发现需要首先创建一个Query 实例，表示是一个查询动作。query 对象，继续补充一个Criteria 实例。Criteria 英\[kraɪ'tɪəriə] 译为比标准、准则、尺度。我们可以直接理解为查询条件。注意Criteria 实例是由 Criteria.where 方法创建出来的。(防盗连接：本文首发自https://github.com/jilodream/ )这是一个简单工厂，参数为要查询的表的列名（文档的字段）。再跟一个in() ，表示列的in 操作，in() 中跟的是in操作的值。最后直接用mongoTemplate 实例执行find 操作就好，条件为查询逻辑和表对应的类文件。我们这里使用的是in 操作，除此之外，常用的还有




| 方法 | 作用 | 类比sql |
| --- | --- | --- |
| gt | 表示 大于 | \> |
| gte | 表示 大于等于 | \>\= |
| lt | 表示 小于 | \< |
| lte | 表示 小于等于 | \<\= |
| ne | 表示 不等于 | !\= |
| nin | 表示 不属于 | not in |
| is | 表示等于 | \= |
| regex | 表示 like (注意后面跟正则表达式,如 "^.\*" \+ queryKeyWord \+ ".\*$") | like ‘%关键字%’ |


这些都是基本操作，有sql经验的同学肯定明白具体怎么使用。我们再补充一个模糊查询的例子：




```
1     private void simpleRegexQuery() {
2         Query query = new Query();
3         String queryKeyWord = "ong";
4         query.addCriteria(Criteria.where("name").regex("^.*" + queryKeyWord + ".*$"));
5         List students = mongoTemplate.find(query, Student.class);
6         log.warn("the query result is: {}", students);
7     }
```


输出如下：




```
2024-12-03 14:46:23.781  WARN 81708 --- [           main] c.e.demo.learn.mongo.MongodbController   : the query result is: [Student(_id=674d8152c993425aaa5c4fe9, name=zhonga, description=perfect2, classNo=2, age=15), Student(_id=674d8153c993425aaa5c4fea, name=zhongb, description=perfect2, classNo=2, age=13), Student(_id=674d8153c993425aaa5c4feb, name=zhongc, description=perfect2, classNo=2, age=15), Student(_id=674d8153c993425aaa5c4fec, name=zhongd, description=perfect2, classNo=3, age=15)]
```


（2）接着来看一个相对复杂点的组合条件：两个或条件，类似于SQL中的： A表达式 OR B表达式，代码如下




```
 1     private void simpleOrQuery() {
 2         Query query = new Query();
 3         String queryKeyWord = "ong";
 4         Criteria neCri = Criteria.where("age").ne(15);
 5         Criteria regexCri = Criteria.where("name").regex("^.*" + queryKeyWord + ".*$");
 6         Criteria orCri = new Criteria().orOperator(neCri, regexCri);
 7         query.addCriteria(orCri);
 8         List students = mongoTemplate.find(query, Student.class);
 9         log.warn("the query result is: {}", students);
10     }
```


执行效果如下：




```
2024-12-03 14:48:28.787  WARN 83804 --- [           main] c.e.demo.learn.mongo.MongodbController   : the query result is: [Student(_id=674d8125bf8e9e35bbc08719, name=xiaob, description=good2, classNo=1, age=13), Student(_id=674d8152c993425aaa5c4fe9, name=zhonga, description=perfect2, classNo=2, age=15), Student(_id=674d8153c993425aaa5c4fea, name=zhongb, description=perfect2, classNo=2, age=13), Student(_id=674d8153c993425aaa5c4feb, name=zhongc, description=perfect2, classNo=2, age=15), Student(_id=674d8153c993425aaa5c4fec, name=zhongd, description=perfect2, classNo=3, age=15), Student(_id=674d81dd8130705614f23312, name=bigb, description=nice, classNo=3, age=13)]
```


我们创建好两个Criteria的简单条件之后，再创建一个新的Criteria 实例，用一个or操作将二者关联起来.query 接收最新的Criteria 实例，然后执行查询即可。这里的写法类似于sql中的 where name like "%ong%" or age !\= 15


如果是两个AND 条件，类似于SQL中的： (防盗连接：本文首发自https://github.com/jilodream/ )A表达式 AND B表达式，用法和or的使用方法是一样的 。这里就不举例了，我们这里写一个复杂的用法：




```
 1     private void complexQuery() {
 2         Query query = new Query();
 3 
 4         String queryKeyWord = "ong";
 5         Criteria ageCri = Criteria.where("age").ne(15);
 6         Criteria nameCri = Criteria.where("name").regex("^.*" + queryKeyWord + ".*$");
 7         Criteria cri1 = new Criteria().orOperator(ageCri, nameCri);
 8 
 9         Criteria descCri = Criteria.where("description").is("nice");
10         Criteria classNoCri = Criteria.where("classNo").in(1, 2, 3);
11         Criteria cri2 = new Criteria().andOperator(descCri, classNoCri);
12 
13         query.addCriteria(new Criteria().orOperator(cri1, cri2));
14         List students = mongoTemplate.find(query, Student.class);
15         log.warn("the query result is: {}", students);
16     }
```


输出如下：




```
2024-12-03 14:51:46.908  WARN 92840 --- [           main] c.e.demo.learn.mongo.MongodbController   : the query result is: [Student(_id=674d8125bf8e9e35bbc08719, name=xiaob, description=good2, classNo=1, age=13), Student(_id=674d8152c993425aaa5c4fe9, name=zhonga, description=perfect2, classNo=2, age=15), Student(_id=674d8153c993425aaa5c4fea, name=zhongb, description=perfect2, classNo=2, age=13), Student(_id=674d8153c993425aaa5c4feb, name=zhongc, description=perfect2, classNo=2, age=15), Student(_id=674d8153c993425aaa5c4fec, name=zhongd, description=perfect2, classNo=3, age=15), Student(_id=674d81dc8130705614f23311, name=biga, description=nice, classNo=3, age=15), Student(_id=674d81dd8130705614f23312, name=bigb, description=nice, classNo=3, age=13), Student(_id=674d81dd8130705614f23313, name=bigc, description=nice, classNo=3, age=15), Student(_id=674d81dd8130705614f23314, name=bigd, description=nice, classNo=3, age=15)](防盗连接：本文首发自https://github.com/jilodream/ )
```


 这里的写法类似于sql中的


where ( description \= "nice" and classNo in (1 ,2 ,3\) ) or ("name like %ong%" or age !\= 15\) 


总体来看：一个Criteria 实例，就是一个查询条件。


我们可以通过 or、and 操作来不断的组合生成一个新的Criteria实例，也就是一个新的查询条件 ，并且可以以此查询条件继续组合生成更高级的Criteria，以此不断的类推。


这个过程就像垒积木一样：


*![](https://img2024.cnblogs.com/blog/704073/202412/704073-20241203145440527-895795380.jpg)*


(3\) 接着我们整合下分页功能，并且以班级排序




```
PageRequest pageable = PageRequest.of(pageIndex - 1, pageSize);
Query pageQuery=query.with(pageable).with(Sort.by(Sort.Direction.DESC,"classNo"));
```


注意分页时，页码数是从0开始，所以要\-1。同时排序使用Sort生成sort的对象，包含排序方式和字段，并且这里支持多级排序。


整体代码如下：




```
 1     private void complexPageQuery() {
 2         int pageIndex=2;
 3         int pageSize=3;
 4         Query query = new Query();
 5         String queryKeyWord = "ong";
 6         Criteria ageCri = Criteria.where("age").ne(15);
 7         Criteria nameCri = Criteria.where("name").regex("^.*" + queryKeyWord + ".*$");
 8         Criteria cri1 = new Criteria().orOperator(ageCri, nameCri);
 9 
10         Criteria descCri = Criteria.where("description").is("nice");
11         Criteria classNoCri = Criteria.where("classNo").in(1, 2, 3);
12         Criteria cri2 = new Criteria().andOperator(descCri, classNoCri);
13 
14         query.addCriteria(new Criteria().orOperator(cri1, cri2));
15         long allDataSize = mongoTemplate.count(query, Student.class);
16         PageRequest pageable = PageRequest.of(pageIndex - 1, pageSize);
17         Query pageQuery=query.with(pageable).with(Sort.by(Sort.Direction.DESC,"classNo"));
18         List students = mongoTemplate.find(pageQuery, Student.class);
19         log.warn("the query result is: {}", students);
20     }
```


输出如下：




```
2024-12-03 14:56:46.059  WARN 18516 --- [           main] c.e.demo.learn.mongo.MongodbController   : the query result is: [Student(_id=674d81dd8130705614f23314, name=bigd, description=nice, classNo=3, age=15), Student(_id=674d81dc8130705614f23311, name=biga, description=nice, classNo=3, age=15), Student(_id=674d8152c993425aaa5c4fe9, name=zhonga, description=perfect2, classNo=2, age=15)]
```


 


 本博客参考[豆荚加速器官网](https://baitenghuo.com)。转载请注明出处！
