## Vector 处理kafka中日志并转发给loki-02

### 部署minio



minio部署要求：

1. 如果是集群部署的话，所有的节点必须有一块独立的磁盘用于存放minio的数据。不能与与'/'分区或其他分区在同一硬盘上。否则会报错。`Disk `   `http://rnode4:9000/data1/export1` `   ` `is part of root disk, will not be used (*errors.errorString)`

### 基于minio部署loki

