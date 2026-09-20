
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