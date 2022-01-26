### 一、网上使用方法
#### 1. 查看优化器状态
   show variables like 'optimizer_trace';
#### 2. 会话级别临时开启
   set session optimizer_trace="enabled=on",end_markers_in_json=on;
#### 3. 设置优化器追踪的内存大小
   set OPTIMIZER_TRACE_MAX_MEM_SIZE=1000000;
#### 4. 执行自己的SQL
   select host,user,plugin from user;
#### 5. information_schema.optimizer_trace表
   SELECT trace FROM information_schema.OPTIMIZER_TRACE;
#### 6. 导入到一个命名为xx.trace的文件，然后用JSON阅读器来查看
   SELECT TRACE INTO DUMPFILE "/tmp/test.trace" FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
   > 注意：（如果没有控制台权限，或直接交由运维，让他把该 trace 文件，输出给你就行了。）。

**注意：不设置优化器最大容量的话，可能会导致优化器返回的结果不全。**

### 二、msyql官网使用方式
``` sql
# Turn tracing on (it's off by default):
SET optimizer_trace="enabled=on";
SELECT ...; # your query here
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
# possibly more queries...
# When done with tracing, disable it:
SET optimizer_trace="enabled=off";
```

A session can trace only statements which it executes; it cannot see a trace of another session.