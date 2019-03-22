title: group by关于sql_mode=only_full_group_by报错
categories: mysql
---

### group by作用

根据指定数段对查询结果进行分组

### sql_mode=only_full_group_by报错

对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY从句中，也就是说查出来的列必须在group by后面出现否则就会报错，或者这个字段出现在聚合函数里面。

解决方法：关闭 only_full_group_by 模式或者使用any()或者其它聚合函数作用于不在group by后的字段

```
set @@sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
```

