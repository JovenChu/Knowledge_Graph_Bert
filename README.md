# Knowledge_Gragh_Bert
The recurrent of KBQA by using Mysql to storage the triple data, using Bert for NER and Sentence Similarity

1. 需要下载BERT的中文预训练模型配置文件`Chinese_L-12_H-768_A-12`到`ModelParams`文件夹中。
2. Data文件夹存放原始数据和处理好的数据
   * `python construct_dataset.py` ：生成NER_Data的数据
   * `python construct_dataset_attribute.py` ：生成Sim_Data的数据
   * `python triple_clean.py` ：生成三元组数据

3. 服务器创建mysql数据库：

   * 安装msql：

     ```shell
     $ sudo apt-get update
     $ sudo apt-get install mysql-server mysql-client libmysqlclient-dev
     ```

   * 安装后查看是否安装成功

     ```shell
     $ sudo netstat -tap | grep mysql
     ```

   * 查看`load_dbdata.py`代码中包含的数据库信息：

     * user="root",
     * password="123456",
     * host="127.0.0.1",
     * port=3306,
     * db="KB_QA",
     * charset="utf8"

   * 使用root用户登录，创建数据库，创建用户并授权

     ```shell
     # root为你的root名，yourpassword为你root用户的密码
     $ mysql -uroot -p123456
     # testDB为创建的数据库名
     $ mysql>create database KB_QA;
     # 退出
     $ mysql>quit;
     ```

   * 数据库信息查看命令：https://www.cnblogs.com/yangyuqiu/p/6391395.html

     * 查看端口号：`show global variables like 'port';`
     * https://www.linuxidc.com/Linux/2018-05/152413.htm

   * 问题解决：

     * 安装完成后出现问题

       ERROR 1698 (28000): Access denied for user 'root'@'localhost'

       * 错误原因：可能是因为初始密码为空；按空格回车后还是报一样的错

       * 解决方案：这时你需要mysql提供给你的密码，在终端输入`sudo vim /etc/mysql/debian.cnf 

       * 获得默认的password：

         ![image-20191009161458310](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u18h7whgj30b704bmxj.jpg)

       * 再使用`mysql -udebian-sys-maint -p`进入mysql后，输入上面获取的密码。

       * 结合两者按此步骤：

         * https://blog.csdn.net/u012804180/article/details/80801351
         * https://www.cnblogs.com/huangguojin/p/9510903.html

       * 重置root账号的密码：

         ```shell
         $ use mysql;
         $ update user set authentication_string=PASSWORD("123456") where User='root';
         ```

       * 更新缓存密码和权限：

         ```shell
         $ update user set plugin="mysql_native_password";
         $ flush privileges;
         $ quit
         $ sudo /etc/init.d/mysql stop
         $ sudo kill -9 $(pgrep mysql)
         $ sudo /etc/init.d/mysql start
         ```

   * others

4. Data文件夹存放原始数据和处理好的数据

   * `python load_dbdata.py `：将数据导入mysql db

   * 查看导入的内容：

     ```shell
     # root为你的root名，yourpassword为你root用户的密码
     $ mysql -uroot -p123456
     # 显示数据库
     $ mysql>show databases;
     # 选择数据库
     $ mysql>use KB_QA;
     # 显示表
     $ mysql>show tables;
     # 显示表中的所有数据
     $ mysql>select * from nlpccQA;
     ```

   * 问题报错解决：

     * [error-code-1406-data-too-long-for-column-mysql#](https://stackoverflow.com/questions/15949038/error-code-1406-data-too-long-for-column-mysql#)

       * 报错截图：

         ![image-20191010100332604](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u18qxhv4j30kd0gbgof.jpg)

       * 报错原因：字段的长度不够存放数据

       * 解决方法：

         ```shell
         # root为你的root名，yourpassword为你root用户的密码
         $ mysql -uroot -p123456
         # You can run an SQL query within your database management tool
         $ mysql>SET @@global.sql_mode= '';
         ```

     * 

   * 插入数据成功：

     ![image-20191010104444427](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u18w1ugvj30hw0iawgc.jpg)

   * others

5. 命名实体识别的调参和训练：

   * 调节参数：降低batch_size到32，以防止OOM；num_train_epochs增加到10，得到更好的训练效果

   * 训练：`bash run_ner.sh`

   * 训练结果：

     ![image-20191010115335685](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u18zrywzj30kh04p0tj.jpg)

   * 运行预测：`bash terminal_ner.sh`

     ![image-20191010154458646](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u19401xyj30km07u0to.jpg)

6. 模式选择：

   ```
   - terminal_ner.sh
   do_predict_online=True  NER线上预测
   do_predict_outline=True  NER线下预测
   
   - args.py
   train = True  预训练模型
   test = True  SIM线上测试
   ```

7. 基于Bert的句子相似度计算模块训练：

   * 训练：`python run_similarity.py`

   * 训练结果：

     ![image-20191010123931546](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u198p4frj30kl066gmq.jpg)

8. 基于KB的问答测试：

   * 运行：`python kbqa_test.py`

   * 测试结果：

     * dataset_test测试：用训练问答对中的实体+属性，去知识库中进行问答测试准确率上限

       ![image-20191010162751913](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u19bzbvuj30kh02bglv.jpg)

     * kb_fuzzy_classify_test测试：

       * 进行问答测试：

         * 1、 实体检索:输入问题，ner得出实体集合，在数据库中检索与输入实体相关的所有三元

         * 2、 属性映射——bert分类/文本相似度

           - 非语义匹配：如果所得三元组的关系(attribute)属性是 输入问题 字符串的子集，将所得三元组的答案(answer)属性与正确答案匹配，correct +1

           - 语义匹配：利用bert计算输入问题(input question)与所得三元组的关系(attribute)属性的相似度，将最相似的三元组的答案作为答案，并与正确的答案进行匹配，correct +1

         * 3、 答案组合    

       * 结果：

         ![image-20191011094817373](https://tva1.sinaimg.cn/large/006y8mN6gy1g7u19falb8j30kl0163yl.jpg)

9. Original artical: https://github.com/WenRichard/KBQA-BERT