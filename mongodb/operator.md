### 存在性查询 { field: { $exists: <boolean> } }

### \$or, \$and, \$nor no empty array 不能传空数组
  ```
  很大的一个坑...清空测试数据库后, 启nest, 发现启不动, 折腾了半天发现某个module挂了
  个onApplicationBootstrap钩子函数, 做个了空数组查询,搞挂了整个app,这个错误由于是mongo内部捕捉的,并无外部调用栈信息,出现了可有的查了.
  ```
