
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