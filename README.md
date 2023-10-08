# 在线机器学习demo

## 前言

使用的flink为1.14.4版本，java为1.8开源版

参考文档为：

`https://nightlies.apache.org/flink/flink-docs-release-1.14/zh/docs`

参考项目为：
`https://github.com/uncleguanghui/pyflink_learn`

主要涉及的知识点为

1. flink 从kafka建表（不断向kafka发送数据）
2. flink 创建udf函数，将表中的每条数据进行训练，得到新的模型发送到Redis
3. 通过flink一个查询指令调用udf函数达到‘在线机器学习的目的’
4. 使用flask构建一个后端应用，将用户在网页中书写的图片调用模型进行推理


注：此项目仅为demo，要扩大数据量，以及拓展应用场景都需要更多的努力
    模型选择为SGDClassifier，是一个基础的轻量级的模型

## 基础服务部署


### docker部署

kafka 和 redis

`docker-compose.yml`中可能需要注意的：

1. kafka的主机地址，写localhost和127.0.0.1的话，远程连接会有问题
2. Redis的密码，需要和程序同步

### 本机部署

flink 和 python

python 环境 我使用conda pack导出，较大，不上传

`mkdir online-learning && tar -zxvf online-learning.tar.gz -C online-learning/`
flink 本地 standalone模式

## 启动项目

1. 启动flink集群 和 基础组件
   `bash flink/bin/start-cluster.sh`
   `docker compose up -d`

2. 向kafka发送数据
   `nohup python kafka_producer.py  >logs/kafka_producer.out 2>&1 &`

3. 提交flink作业
   a. `export PYFLINK_CLIENT_EXECUTABLE={conda_env_name}/bin/python`
   或者使用命令行参数-pyclientexec 这个参数指定的python解释器负责提交job，编译udf的工作
   详见`https://nightlies.apache.org/flink/flink-docs-release-1.14/zh/docs/dev/python/python_config/#python-client-executable`
   
   b. -pyexec指定的python解释器负责udf的执行

   如果本地不执行pip install安装python依赖，那么需要指定以上环境变量以及指定-pyexec.
   `flink run -m localhost:8081 -py stream.py -pyexec $(which python)`
   
   `-pyexec`指定python解析器,使用导出的conda环境的话，指定`{conda_env_name}/bin/python`即可

   举个例子：
   `tar -xzvf online-learning.tar.gz -C online-learning` # 将压缩好的python环境解压到online-learning文件夹中
   `flink run -m localhost:8081 -py stream.py -pyexec online-learning/bin/python`

   使用压缩好的python环境可能无法运行此项目的网页测试程序，根据提示在宿主机上下载相关依赖即可。

5. 打开测试的网页服务

   `nohup python model_server.py  >logs/model_server.out 2>&1 &`

如果打开浏览器进行测试有推理结果的话说明成功。


## 注意事项

1. `templates/web.html`文件的44行需要修改成后端地址

2. `stream.py`需要修改对应的kafka配置和redis配置，在10行，50行左右

3. `model_server.py`修改对应的redis配置

4. `kafka_producer.py`修改对应的kafka配置

5. 提交job之后，终端会被占用，此时Ctrl + C 终止并不会影响job的运行，停止job需要去网页端取消
