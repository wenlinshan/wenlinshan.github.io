LCN如何实现分布式事务

# 1、简介

​		在单体应用当中，只需要一个数据库的情况，数据库有自己的事务，开发人员可以选择默认的数据库的事务，以及选择相应的事务隔离级别之后，开发人员无需考虑当操作多张表发生异常的时候数据一致性的问题。

​		但是在分布式事务当中，每个服务都有自己的数据库，当数据操作设计到多个数据库的时候就无法保证多个服务涉及到的数据能保证一致性，所以这个时候就需要引入第三方组件来实现多个服务之间数据操作的一致性的问题。



