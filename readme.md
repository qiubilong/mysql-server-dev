
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
 ### storage\innobase\row\row0sel.cc:row_search_mvcc
````sql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  `b` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

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

select * from t1 where id = 100;                       从上到下定位游标
select * from t1 where id < 100;                       从左到右遍历 -- 升序       
select * from t1 where id > 100;                       从上到下定位 -- 升序 
select * from t1;                                      从左到右遍历 -- 升序
select * from t1 order by id asc;                      从左到右遍历 -- 升序
select * from t1 order by id desc;                               从右到左遍历 -- 降序
select * from t1 where id < 100 order by id desc;                从上到下定位 -- 降序
select * from t1 where id < 100 order by id asc;       从左到右遍历 -- 升序
select * from t1 where id > 100 order by id asc;       从上到下定位 -- 升序 
select * from t1 where id > 100 order by id desc;                 从右到左遍历 -- 降序
select * from t1 where id in (6,3,4);

````