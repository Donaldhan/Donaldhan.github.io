

# 集群配置的注意点
Other useful configuration parameters that you can customize include:

HADOOP_PID_DIR - The directory where the daemons’ process id files are stored.
HADOOP_LOG_DIR - The directory where the daemons’ log files are stored. Log files are automatically created if they don’t exist.
HADOOP_HEAPSIZE / YARN_HEAPSIZE - The maximum amount of heapsize to use, in MB e.g. if the varibale is set to 1000 the heap will be set to 1000MB. This is used to configure the heap size for the daemon. By default, the value is 1000. If you want to configure the values separately for each deamon you can use.


另外一个需要注意点是，数据目录最好存储在指定的目录，默认存储在临时的文件夹内，防止重启系统时，hdfs文件数据丢失。

In most cases, you should specify the HADOOP_PID_DIR and HADOOP_LOG_DIR directories such that they can only be written to by the users that are going to run the hadoop daemons. Otherwise there is the potential for a symlink attack.


# 搭建idea-hadoop开发环境


## idea intellij 连接hadoopHDFS插件
https://blog.csdn.net/kismet2399/article/details/85090839

#工程创建
https://blog.csdn.net/chaoping315/article/details/78904970

https://www.cnblogs.com/YellowstonePark/p/7699083.html

从hadoop安装的example中，添加示例。
