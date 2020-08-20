| 事务隔离级别               | 脏读   | 不可重复读 | 幻读   | 加锁方案       |
| -------------------------- | ------ | ---------- | ------ | -------------- |
| 未提交读(Read Uncommitted) | 可能   | 可能       | 可能   | 读取时加共享锁 |
| 已提交读(Read Committed)   | 不可能 | 可能       | 可能   | 读取时加排他锁 |
| 可重复读(Repeatable Read)  | 不可能 | 不可能     | 可能   |                |
| 序列化读(Serializable)     | 不可能 | 不可能     | 不可能 |                |

#### 1 未提交读

脏读示例：

| 时间 | 事务A                               | 事务B                                         |
| ---- | ----------------------------------- | --------------------------------------------- |
| 1    | BEGIN                               | BEGIN                                         |
| 2    | SELECT * FROM student WHERE id = 1; |                                               |
| 3    |                                     | UPDATE student SET score = '90' WHERE id = 1; |
| 4    | SELECT * FROM student WHERE id = 1; |                                               |
| 5    | COMMIT                              | COMMIT                                        |

#### 2 已提交读

不可重复读示例：


| 时间 | 事务A                     | 事务B                |
| ---- | ------------------------ | ----------------------- |
| 1    | BEGIN                    | BEGIN                  |
| 2    | SELECT * FROM student WHERE id = 1; |                        |
| 3    |              | UPDATE student SET score = '90' WHERE id = 1; |
| 4    |              | COMMIT                                        |
| 5    | SELECT * FROM student WHERE id = 1; |                        |
| 6    | COMMIT                              |                        |

#### 3 可重复读

幻读示例：

<table>
  <tr><th>时间</th><th>事务A</th><th>事务B</th></tr>
  <tr>
      <td>1</td>
      <td>BEGIN</td>
      <td>BEGIN</td>
  </tr>
  <tr>
      <td>2</td>
      <td>SELECT * FROM student WHERE id = 1 FOR UPDATE</td>
      <td></td>
  </tr>
  <tr>
      <td>3</td>
      <td></td>
      <td>UPDATE student SET score = '90' WHERE id = 1;</td>
  </tr>
  <tr>
      <td>4</td>
      <td></td>
      <td>COMMIT</td>
  </tr>
  <tr>
      <td>5</td>
      <td>SELECT * FROM student WHERE id = 1;</td>
      <td></td>
  </tr>
  <tr>
      <td>6</td>
      <td>COMMIT</td>
      <td></td>
  </tr>
</table> 


#### 4 序列化读

