## Docker镜像迁移

#### docker镜像迁移

1. 使用save

   ```bash
   $ docker save reg.example.com/business/underwriting_ai_package_main:v1.2.2>/usr/local/src/underwriting_ai_package_main.tar
   ```

2. 使用zip进行压缩

   ```bash
   $ zip underwriting_ai_package_main.zip underwriting_ai_package_main.tar
   $ zip -s 3072m underwriting_ai_package_main.zip --out underwriting_ai_package_main_sub.zip
   ```

3. 然后拷贝过去，再进行合并

   ```bash
   $ cat underwriting_ai_package_main_sub.z* > underwriting_ai_package_main.zip
   $ unzip underwriting_ai_package_main.zip
   ```

4. 然后load

   ```bash
   $ docker load<underwriting_ai_package_main.tar
   ```

   



export和import命令

```bash
$ docker export 98ca36> ubuntu.tar
$ cat ubuntu.tar | docker import - ubuntu:import
```

export 和 import 导出的是一个容器的快照, 不是镜像本身, 也就是说没有 layer.

你的 dockerfile 里的 workdir, entrypoint 之类的所有东西都会丢失，commit 过的话也会丢失。

快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也更大。

-  docker save 保存的是镜像（image），docker export 保存的是容器（container）；
-  docker load 用来载入镜像包，docker import 用来载入容器包，但两者都会恢复为镜像；
-  docker load 不能对载入的镜像重命名，而 docker import 可以为镜像指定新名称。



#### zip分割压缩和合并解压

```
# 准备工作：将文件或文件夹打包为zip压缩包
zip -r src.zip ./src

1. 分卷压缩
# 压缩后src.zip为4.6G，将其分割，每个子压缩包不超过1G，生成5个压缩包src_split.z01（1G）、src_split.z02（1G）、src_split.z03（1G）、src_split.z04（1G）和src_split.zip（0.6G）
zip -s 1024m src.zip --out src_split.zip
-s: 创建分卷的大小
-r: 循环压缩文件夹下面的内容
2. 合并解压（方法1）
# 将上述5个压缩包合并为一个压缩文件single.zip
zip src_split.zip -s=0 --out single.zip
# 解压single.zip
unzip -d ./single.zip

3、合并解压（方法2）
# 例如将linux.zip文件夹压分割为：linux.zip.001, linux.zip.002, linux.zip.003, ... 则：
首先 cat linux.zip* > linux.zip  #合并为一个zip包
然后 unzip linux.zip #解压zip包
-----------------------------------
zip/tar 分割压缩和合并解压
```

#### tar分割压缩和合并解压

```
# 准备工作：打包压缩文件
tar -zcvf src.tar.gz ./src
#如果待压缩的文件夹中包含软链接或者硬链接，需要将其指向的文件(夹)也打包进去的话，需要加上参数-h,即
tar -zcvfh src.tar.gz ./src

注：如果只想打包，不想压缩，可以将参数z去除，即：tar -cvf imgs.tar ./imgs

1. 解压文件
tar -zxvf src.tar.gz
#解压到指定目录tmp
tar -zxvf src.tar.gz -C ./tmp

2.分割大文件,每个文件最大100M
2.1)分割为每个子压缩包不超过100M
split -b 100m src.tar.gz src.tar.gz

2.2)后缀设为两位数字
//-d 制定生成的分割包后缀为数字形式，-a 1 设定序列的长度为1（默认值为2）
split -a 2 -d -b 100m imgs.tar.gz imgs.tar.gz

3.合并文件
cat imgs.tar.gz.* > imgs.tar.gz

4. 打包压缩并分割大文件
tar -czvf - ./src| split -a 2 -d -b 100m - src.tar.gz

6. 合并并解压文件
cat src.tar.gz.* | tar-zxvf -
-----------------------------------
zip/tar 分割压缩和合并解压

```



参考链接：

https://blog.51cto.com/u_12461028/5454535

https://www.runoob.com/docker/docker-save-command.html



