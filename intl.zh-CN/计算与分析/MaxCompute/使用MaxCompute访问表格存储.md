# 使用MaxCompute访问表格存储 {#concept_omw_smj_bfb .concept}

## 背景信息 {#section_u2y_dqh_dfb .section}

本文介绍如何在同一个云账号下实现表格存储和 MaxCompute 之间的无缝连接。

[MaxCompute](https://www.alibabacloud.com/product/maxcompute)是一项大数据计算服务，它能提供快速、完全托管的 PB 级数据仓库解决方案，使您可以经济并高效地分析处理海量数据。您只需通过一条简单的 DDL 语句，即可在 MaxCompute 上创建一张外部表，建立 MaxCompute 表与外部数据源的关联，提供各种数据的接入和输出能力。MaxCompute 表是结构化的数据，而外部表可以不限于结构化数据。

表格存储与 MaxCompute 都有其自身的类型系统，两者之间的类型对应关系如下表所示。

|Table Store|MaxCompute|
|:----------|:---------|
|STRING|STRING|
|INTEGER|BIGINT|
|DOUBLE|DOUBLE|
|BOOLEAN|BOOLEAN|
|BINARY|BINARY|

## 准备工作 {#section_jgw_pqh_dfb .section}

使用 MaxCompute 访问表格存储前，您需要完成以下准备工作：

1.  开通[MaxCompute服务](https://www.alibabacloud.com/product/maxcompute)。
2.  创建[MaxCompute项目](https://www.alibabacloud.com/help/doc-detail/30263.htm)。
3.  创建[AccessKey](https://www.alibabacloud.com/help/doc-detail/53045.htm)。
4.  创建[AccessKey](https://partners-intl.aliyun.com/help/product/53045.htm)。
5.  在 RAM 控制台授权 MaxCompute 访问表格存储的权限。
    -   方法一：登录阿里云账号[单击此处完成一键授权](https://ram.console.aliyun.com)。

    -   方法二：进行手动授权。步骤如下：

        1.  登录 [RAM 控制台](https://ram.console.aliyun.com/?spm=a2c4g.11186623.2.20.447730b3NxdBQA)。
        2.  在角色管理页面，创建用户角色 AliyunODPSDefaultRole。

            ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20327/153968017011954_zh-CN.png)

        3.  在角色详情页面，设置策略内容。

            ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20327/153968017011955_zh-CN.png)

            策略内容如下：

            ```
            {
            "Statement": [
            {
            "Action": "sts:AssumeRole",
            "Effect": "Allow",
            "Principal": {
             "Service": [
               "odps.aliyuncs.com"
             ]
            }
            }
            ],
            "Version": "1"
            }
            ```

        4.  在策略管理页面，新建授权策略 AliyunODPSRolePolicy。

            ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20327/153968017011956_zh-CN.png)

            策略内容如下：

            ```
            {
            "Version": "1",
            "Statement": [
             {
               "Action": [
                 "ots:ListTable",
                 "ots:DescribeTable",
                 "ots:GetRow",
                 "ots:PutRow",
                 "ots:UpdateRow",
                 "ots:DeleteRow",
                 "ots:GetRange",
                 "ots:BatchGetRow",
                 "ots:BatchWriteRow",
                 "ots:ComputeSplitPointsBySize"
               ],
               "Resource": "*",
               "Effect": "Allow"
             }
            ]
            }
            --还可自定义其他权限
            ```

        5.  在角色管理页面，将 AliyunODPSRolePolicy 权限授权给 AliyunODPSDefaultRole 角色。

            ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20327/153968017111957_zh-CN.png)

6.  在表格存储控制台[创建实例](../../../../intl.zh-CN/快速入门/创建实例.md#)和[创建数据表](../../../../intl.zh-CN/快速入门/创建数据表.md#)。

    在本示例中，创建的表格存储实例和数据表信息如下：

    -   实例名称：cap1
    -   数据表名称：vehicle\_track
    -   主键信息：vid \(integer\)，gt \(integer\)
    -   访问域名：`https://cap1.cn-hangzhou.ots-internal.aliyuncs.com`

        **说明：** 使用 MaxCompute 访问表格存储时，建议使用表格存储的私网地址。

    -   设置实例网络类型为**允许任意网络访问**。

        ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20327/153968017111958_zh-CN.png)


## 步骤 1. 安装并配置客户端 {#section_v4j_ssh_dfb .section}

1.  下载 [MaxCompute 客户端](http://repo.aliyun.com/download/odpscmd/latest/odpscmd_public.zip?spm=a2c4g.11186623.2.28.447730b3NxdBQA&file=odpscmd_public.zip)并解压。

    **说明：** 确保您的机器上已安装 JRE 1.7或以上版本。

2.  编辑 conf/odps\_config.ini 文件，对客户端进行配置，如下所示：

    ```
    access_id=*******************
    access_key=*********************
     # Accesss ID 及 Access Key 是用户的云账号信息，可登录阿里云官网，进入管理控制台 — accesskeys 页面进行查看。
    project_name=my_project
     # 指定用户想进入的项目空间。
    end_point=https://service.odps.aliyun.com/api
     # MaxCompute 服务的访问链接
    tunnel_endpoint=https://dt.odps.aliyun.com
     # MaxCompute Tunnel 服务的访问链接
    log_view_host=http://logview.odps.aliyun.com
     # 当用户执行一个作业后，客户端会返回该作业的 LogView 地址。打开该地址将会看到作业执行的详细信息
    https_check=true
     #决定是否开启 HTTPS 访问
    ```

    **说明：** odps\_config.ini 文件中使用`#`作为注释，MaxCompute 客户端内使用`--`作为注释。

3.  运行 bin/odpscmd.bat，输入 `show tables；`。

    如果显示当前 MaxCompute 项目中的表，则表述上述配置正确。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/20327/153968017111959_zh-CN.png)


## 步骤 2. 创建外部表 {#section_qkd_hth_dfb .section}

创建一张 MaxCompute 的数据表（ots\_vehicle\_track）关联到 Table Store 的某一张表（vehicle\_track）。

关联的数据表信息如下：

-   实例名称：cap1
-   数据表名称：vehicle\_track
-   主键信息：vid \(int\); gt \(int\)
-   访问域名：`https://cap1.cn-hangzhou.ots-internal.aliyuncs.com`

```
CREATE EXTERNAL TABLE IF NOT EXISTS ots_vehicle_track
(
vid bigint,
gt bigint,
longitude double,
latitude double,
distance double ,
speed double,
oil_consumption double
)
STORED BY 'com.aliyun.odps.TableStoreStorageHandler' -- (1)
WITH SERDEPROPERTIES ( -- (2)
'tablestore.columns.mapping'=':vid, :gt, longitude, latitude, distance, speed, oil_consumption', -- (3)
'tablestore.table.name'='vehicle_track' -- (4)
)
LOCATION 'tablestore://cap1.cn-hangzhou.ots-internal.aliyuncs.com'; -- (5)
```

参数说明如下：

|标号|参数|说明|
|:-|:-|:-|
|（1）|com.aliyun.odps.TableStoreStorageHandler|MaxCompute 内置的处理 Table Store 数据的 StorageHandler，定义了 MaxCompute 和 Table Store 的交互，相关逻辑由 MaxCompute 实现。|
|（2）|SERDEPROPERITES|可以理解为提供参数选项的接口，在使用 TableStoreStorageHandler 时，有两个必须指定的选项，分别是 tablestore.columns.mapping 和 tablestore.table.name。|
|（3）|tablestore.columns.mapping|必填选项。MaxCompute 将要访问的 Table Store 表的列，包括主键和属性列。 其中，带`:`的表示 Table Store 主键，例如本示例中的 :vid 与 :gt，其他均为属性列。在指定映射的时候，用户必须提供指定 Table Store 表的所有主键，属性列无需全部提供，可以只提供需要通过 MaxCompute 来访问的属性列。|
|（4）|tablestore.table.name|需要访问的 Table Store 表名。 如果指定的 Table Store 表名错误（不存在），则会报错。MaxCompute 不会主动创建 Table Store 表。|
|（5）|LOCATION|指定访问的 Table Store 的实例信息，包括实例名和 endpoint 等。|

## 步骤 3. 通过外部表访问 Table Store 数据 {#section_tlr_1j3_dfb .section}

创建外部表后，Table Store 的数据便引入到了 MaxCompute 生态中，您可通过 MaxCompute SQL 命令来访问 Table Store 数据。

```
// 统计编号 4 以下的车辆在时间戳 1469171387 以前的平均速度和平均油耗
select vid,count(*),avg(speed),avg(oil_consumption) from ots_vehicle_track where vid <4 and gt<1469171387  group by vid;
```

返回类似如下结果：

![](images/11960_zh-CN.jpeg)

