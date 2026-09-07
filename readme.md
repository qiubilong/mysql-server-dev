
## 创建表
storage\innobase\handler\ha_innodb.cc:create_table()
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