# 使用vacuum命令重建Sqlite数据库文件空间列表
在使用SQlite数据库时，当插入大量数据后，文件会不断增长，但删除这些数据后，数据库文件不会释放磁盘空间。
>SQlite官网有[解释](https://www.sqlite.org/lang_vacuum.html)

如果要释放数据库文件空闲的磁盘空间，可手动执行命令：
```c++
	db->exec("DELETE FROM table");
	db->exec("vacuum"); // 释放空闲的磁盘空间，注：无法恢复被删除的数据
```
