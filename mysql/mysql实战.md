# mysql实战

------


## 唯一索引和普通索引
**查询过程**
select id from T where k=5。这个查询语句在索引树上查找的过程，先是通过 B+ 树从树根开始，按层搜索到叶子节点，然后可以认为数据页内部通过二分法来定位记录。
 - 对于普通索引来说，查找到满足条件的第一个记录 (5,500) 后，需要查找下一个记录，直到碰到第一个不满足 k=5 条件的记录。
 - 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。
结论：查询性能微乎其微
**更新过程**
 - change buffer
 当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InnoDB 会将这些更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了。
唯一索引的更新就不能使用 change buffer，实际上也只有普通索引可以使用。
change buffer 用的是 buffer pool 里的内存，因此不能无限增大。
更新性能普通索引高于唯一索引，普通索引使用了change buffer。
- change buffer 的使用场景
merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。
**写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时 change buffer 的使用效果最好。这种业务模型常见的就是账单类、日志类的系统**

## 怎么给字符串字段加索引
字符串支持前缀索引
```
alter table SUser add index index1(email);或mysql> alter table SUser add index index2(email(6));
alter table SUser add index index1(email);或mysql> alter table SUser add index index2(email(6));
```
**使用的是 index1**
```
1.从 index1 索引树找到满足索引值是’zhangssxyz@xxx.com’的这条记录，取得 ID2 的值；
2.到主键上查到主键值是 ID2 的行，判断 email 的值是正确的，将这行记录加入结果集；
3.取 index1 索引树上刚刚查到的位置的下一条记录，发现已经不满足 email='zhangssxyz@xxx.com’的条件了，循环结束。
这个过程中，只需要回主键索引取一次数据，所以系统认为只扫描了一行。
```
**使用的是 index2**
```
1.从 index2 索引树找到满足索引值是’zhangs’的记录，找到的第一个是 ID1；
2.到主键上查到主键值是 ID1 的行，判断出 email 的值不是’zhangssxyz@xxx.com’，这行记录丢弃；
3.取 index2 上刚刚查到的位置的下一条记录，发现仍然是’zhangs’，取出 ID2，再到 ID 索引上取整行然后判断，这次值对了，将这行记录加入结果集；
4.重复上一步，直到在 idxe2 上取到的值不是’zhangs’时，循环结束。
在这个过程中，要回主键索引取 4 次数据，也就是扫描了 4 行。
```
**使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。**
**使用前缀索引就用不上覆盖索引对查询性能的优化了，这也是你在选择是否使用前缀索引时需要考虑的一个因素。**

 1. 使用倒序存储 如果你存储身份证号的时候把它倒过来存.
 ```sql
 select field_list from t where id_card = reverse('input_id_card_string');
 ```
 2. 使用 hash 字段  在表上再创建一个整数字段，来保存身份证的校验码，同时在这个字段上创建索引。
  ```sql
 alter table t add id_card_crc int unsigned, add index(id_card_crc);
 select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string'
 ```
 **倒序存储和使用 hash 字段这两种方法的异同点**
 
 1. 从占用的额外空间来看，倒序存储方式在主键索引上，不会消耗额外的存储空间，而 hash 字段方法需要增加一个字段。当然，倒序存储方式使用 4 个字节的前缀长度应该是不够的，如果再长一点，这个消耗跟额外这个 hash 字段也差不多抵消了。
 2. 在 CPU 消耗方面，倒序方式每次写和读的时候，都需要额外调用一次 reverse 函数，而 hash 字段的方式需要额外调用一次 crc32() 函数。如果只从这两个函数的计算复杂度来看的话，reverse 函数额外消耗的 CPU 资源会更小些。
 3. 从查询效率上看，使用 hash 字段方式的查询性能相对更稳定一些。因为 crc32 算出来的值虽然有冲突的概率，但是概率非常小，可以认为每次查询的平均扫描行数接近 1。而倒序存储方式毕竟还是用的前缀索引的方式，也就是说还是会增加扫描行数。