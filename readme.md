
## 创建表 -- 创建索引 -- 构建B+树🌲
### storage\innobase\handler\ha_innodb.cc:create_table()
  ### 构建B+树 storage\innobase\btr\btr0btr.cc:btr_create

一个【索引】两个段（叶子节点段、非叶子结点段），从【段】中申请【空闲页】，【段】管理了【空闲区】链表，一个【区】64页
```` sql
CREATE TABLE `t1` (
`id` int(11) NOT NULL,
`a` int(11) DEFAULT NULL,
`b` varchar(255) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB;
````

## 段申请空闲页
storage\innobase\fsp\fsp0fsp.cc:fseg_alloc_free_page_low

## 溢出字段
查看表空间文件  hexdump -C t.ibd
````sql
create table t(id int primary key,  a varchar(10000) not null);

insert into t values(1, repeat('a',10000));
insert into t values(2,'dadudu');

````

## 查询数据
 ###  读取一行记录 storage\innobase\row\row0sel.cc:row_search_mvcc
       -- 从上到下定位 storage\innobase\include\btr0pcur.ic:btr_pcur_open_with_no_init_func 
       -- 从左到右，从右到左遍历 storage\innobase\btr\btr0cur.cc:btr_cur_open_at_index_side_func
````sql
CREATE TABLE `t1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `a` int(11) DEFAULT NULL,
    `b` varchar(20) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `a` (`a`)
) ENGINE=InnoDB AUTO_INCREMENT=501 DEFAULT CHARSET=utf8mb4;

DELIMITER $$
CREATE PROCEDURE `insert_t1_500`()
BEGIN
    DECLARE i INT DEFAULT 1;

    WHILE i <= 500 DO
        INSERT INTO `t1` (`a`, `b`) VALUES (FLOOR(1 + RAND() * 1000), CONCAT('test_', i));
        SET i = i + 1;
END WHILE;
END$$
DELIMITER ;

CALL insert_t1_500();

------------------------------------------------------------

select * from t1 where id = 100;                       从上到下定位
select * from t1 where id < 100;                       从左到右遍历 -- 升序       
select * from t1 where id > 100;                       从上到下定位 -- 升序 
select * from t1;                                      从左到右遍历 -- 升序
select * from t1 order by id asc;                      从左到右遍历 -- 升序
select * from t1 order by id desc;                               从右到左遍历 -- 降序
select * from t1 where id < 100 order by id desc;                从上到下定位 -- 降序
select * from t1 where id < 100 order by id asc;       从左到右遍历 -- 升序
select * from t1 where id > 100 order by id asc;       从上到下定位 -- 升序 
select * from t1 where id > 100 order by id desc;                 从右到左遍历 -- 降序
select * from t1 where id in (6,3,4);                  从上到下定位


记录之间只有向后指针，尽量升序遍历，降序是迂回前进
````

 ### 回表判断 storage\innobase\handler\ha_innodb.cc:build_template
      -- select id from t1 where a = 317; 辅助索引不包含查询的字段时，需要主键回表 storage\innobase\row\row0sel.cc:row_sel_get_clust_rec_for_mysql
      -- 辅助索引不需要主键回表，那就是索引覆盖
 ### 排序 order by -- 尽量内存排序(256k) > （参与字段长度1k）内存主键排序+回表 > 归并排序临时文件  sql\filesort.cc:filesort --> sql\filesort.cc:find_all_keys
     -- 不走索引，全表扫描后再排序 select * from t1 order by a;
     - 优先在内存sortBuffer排序，sort_buffer_size默认256k，订单库2M。
     - 当参与字段长度>1k时，使用主键排序，再回表 select * from t2 order by a; ---- 需要回表，尽量避免
     - sortBuffer内存用完，先排序写入临时文件，再归并排序有序临时文件  ---- 文件排序要避免
     - 走索引不需要排序
````sql
CREATE TABLE `t2` (  -- t2表可以模拟主键排序再回表
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `a` int(11) DEFAULT NULL,
    `b` varchar(1024) DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=501 DEFAULT CHARSET=utf8mb4;
````

 ### 去重 groupby  内存临时表hash > 【磁盘临时表】 --- 必要时增加max_heap_table_size，尽量内存去重
     -- 非函数计算 sql\sql_executor.cc:end_write   ---例如 select a from t1 group by a;
     -- 函数计算   sql\sql_executor.cc:end_update  ---例如 select a,count(*) from t1 group by a;
     -- 【内存临时表】超过 max_heap_table_size=16M，转换为【磁盘临时表】
     -- mysql8不会再默认排序，mysql5.7需要指定order by null关闭排序

### join连表 sql\sql_executor.cc:sub_select  --> evaluate_join_record 执行where过滤 存入joinBuffer缓存   --> 连接匹配 sql\sql_join_buffer.cc:join_matching_records
    --- 非索引连表，全表扫描t1查出记录，where条件过滤满足 --> 存到 joinBuffer (256k) --> joinBuffer 已满或者t1记录结束 --> 遍历t2表匹配joinBuffer，也就是块循环BNL，减少全表扫描次数，t1（N） * t2（M） ---> t1（N）/ joinBuffer * t2（M）
    --- select * from t1 join t2 on t1.id=t2.id;
        select * from t1 join t2 on t1.id=t2.a where t1.a=200          
        explain 出现Using join buffer,不走索引。 sql\sql_join_buffer.cc:join_matching_records 查询【驱动表记录】存入joinBuffer后， 每读取【被驱动表】一条记录，循环joinBuffer进行匹配

           -- 遍历【驱动表】 sub_select-->循环 【row_search_mvcc --> evaluate_join_record -->where匹配 --> sub_select_op 存储记录到joinBuffer缓存】
             -- 遍历【驱动表】结束、joinBuffer满 --> sub_select_op --> join_matching_records -->  遍历【被驱动表】 --> 匹配 joinBuffer --> 发送结果缓存

    --- 走索引连表，t1查到一行记录，接着走t2索引匹配        -- 性能最佳，看explain 无 Using join buffer。 全表/走索引a 扫描t1 --> where条件过滤满足 --> 走t2表索引 --> where条件过滤满足 --> 发送结果缓存
        select * from t1 join t2 on t1.id=t2.id where t1.a=200;  
            -- 遍历【驱动表】 sub_select-->循环 【row_search_mvcc --> evaluate_join_record -->where匹配 --> sub_select被驱动表】
              -- 遍历【被驱动表】 sub_select-->循环 【id回表row_search_mvcc -->evaluate_join_record-->where匹配-->end_send】

### 子查询
    -- in转连表, 先t2后t1 。例如 select * from t1  where t1.id in ( select id from t2 where a=2 );  
    -- explain 出现【unique_subquery】 子查询，先t1后t2 。例如 select * from t1  where t1.id  not in ( select id from t2 where a=2 );  
    -- in 是集合查询。exist是布尔条件查询，短路操作，符合条件就停止。例如 select * from t1 info  where id=4 and  exists ( select 1 from t2 where t2.id = info.id );  
    -- 先执行内部自查询生成临时表。 例如 select * from  ( select id, a from t2 where a=100 limit 20 ) t  where t.id=4  ;  


### count统计    sub_select  --> 循环 row_search_mvcc --> where过滤 evaluate_join_record -->  累计 sql\item_sum.cc:add
    COUNT(列) 可能走不同索引，COUNT(*)：优化器选最小的索引（通常是二级索引）来数。
    COUNT(id)：如果 id 是主键，走主键，主键索引比二级索引大（包含整行数据），所以 COUNT(id) 可能比 COUNT(*) 慢，因为扫的索引更大
    -- select count(*) from t1;
    -- select count(1) from t1;
    -- select count(id) from t1;
    
### limit截取   sub_select  --> 循环 row_search_mvcc --> where过滤 evaluate_join_record --> 
                       sql\sql_executor.cc:end_send 行数判断 --->  sql\sql_class.cc:send_data 跳过偏移       
                        end_send 方法中 limit结束，返回NESTED_LOOP_QUERY_LIMIT结束外层循环
  

### 插入新记录 sql\sql_insert.cc:mysql_insert --> 多行依次插入 
  - sql\sql_insert.cc:write_record -- insert 、 insert on duplicate update
   - storage\innobase\row\row0ins.cc:row_ins -- 遍历索引，依次插入一行记录
     - storage\innobase\btr\btr0cur.cc:btr_cur_optimistic_insert  -- 乐观插入，当前页空间足够
     - storage\innobase\btr\btr0cur.cc:btr_cur_pessimistic_insert -- 悲观插入，申请新页，页拆分，id自增插入不需【页拆分】
```
要保留历史/追加          → INSERT  --- insert into t1(id,a,b) values (7,70,"test")
冲突时只改几个列         → INSERT ... ON DUPLICATE KEY UPDATE
冲突时整行以新为准(快照)  → REPLACE = delete + insert
冲突时直接跳过不要报错    → INSERT IGNORE
```
```
死锁场景：
 - 两个普通 INSERT 同时复活同一条 delete-marked 记录（S 锁 → 独占 升级成环）
 - 并发 ON DUPLICATE KEY UPDATE 顺序不一样
    T1: INSERT INTO t VALUES (5,1),(9,1) ON DUPLICATE KEY UPDATE v=v+1;
    T2: INSERT INTO t VALUES (9,2),(5,2) ON DUPLICATE KEY UPDATE v=v+1;
 - 两个唯一索引，删除 + insert 死锁
   T1 ODKU：  ① uk('A') 项的 X 记录锁 → ② 请求 行10 的 X 记录锁
   T2 DELETE：③ 行10 的 X 记录锁     → ④ 请求 uk('A') 项的 X 记录锁
 
```

### 删除记录 storage\innobase\row\row0upd.cc:row_upd --> storage\innobase\btr\btr0cur.cc:btr_cur_del_mark_set_clust_rec 设置记录的delete mark
   - purge 线程 1秒异步回收删除记录空间 storage\innobase\row\row0purge.cc:row_purge
       - btr_cur_optimistic_delete 乐观回收 -- 删除后空间利用率<50%
       - btr_cur_pessimistic_delete 悲观回收 -- 删除后空间利用率>50%，需要合并页面
   - 每天集中删百万行 → undo log 暴涨、binlog 大事务 → 主从延迟明显，删除范围持有大量记录锁 + 间隙锁，阻塞并发插入、大量页同时跌破 50% 触发悲观路径（需树 X-latch + 兄弟页 latch），CPU/IO 抖动
   - 方式上永远小批量（这是原则），时段上尽量低峰（这是优化）
   - 删除跌破 50% → 悲观删除 → compress_if_useful → btr_compress 搬记录+锁迁移+摘链表+删父ptr → btr_page_free 归还页