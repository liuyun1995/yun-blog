| 事务隔离级别               | 脏读   | 不可重复读 | 幻读   | 加锁方案       |
| -------------------------- | ------ | ---------- | ------ | -------------- |
| 未提交读(Read Uncommitted) | 可能   | 可能       | 可能   | 读取时加共享锁 |
| 已提交读(Read Committed)   | 不可能 | 可能       | 可能   | 读取时加排他锁 |
| 可重复读(Repeatable Read)  | 不可能 | 不可能     | 可能   |                |
| 序列化读(Serializable)     | 不可能 | 不可能     | 不可能 |                |

#### 1 未提交读

脏读示例：
![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/MySQL%E7%B3%BB%E5%88%97%5B1%5D--%E4%BA%8B%E5%8A%A1%E7%9A%84%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB/1.png)

#### 2 已提交读

不可重复读示例：
![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/MySQL%E7%B3%BB%E5%88%97%5B1%5D--%E4%BA%8B%E5%8A%A1%E7%9A%84%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB/2.png)

#### 3 可重复读

幻读示例：
![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/MySQL%E7%B3%BB%E5%88%97%5B1%5D--%E4%BA%8B%E5%8A%A1%E7%9A%84%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB/3.png)

#### 4 序列化读

