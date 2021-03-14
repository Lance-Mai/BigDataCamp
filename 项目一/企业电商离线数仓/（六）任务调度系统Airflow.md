# 任务调度系统 Airflow

## （一）Airflow简介 

Airflflow 是 Airbnb 开源的一个用 ==**Python**== 编写的调度工具。于 2014 年启动，2015 年春季开源，2016 年加入 Apache 软件基金会的孵化计划。

Airflflow 将一个工作流制定为一组任务的**有向无环图（DAG）**，并指派到一组计算节点上，根据相互之间的依赖关系，有序执行。Airflflow 有以下优势：

-   <u>灵活易用</u>。Airflflow 是 Python 编写的，工作流的定义也使用 Python 编写；

-   <u>功能强大</u>。支持多种不同类型的作业，可自定义不同类型的作业。如 Shell、Python、Mysql、Oracle、Hive等；

-   <u>简洁优雅</u>。作业的定义简单明了；

-   <u>易扩展</u>。提供各种基类供扩展，有多种执行器可供选择；

### 1. 体系架构

![image-20210305153838568](https://raw.githubusercontent.com/LanceMai/MyPictureBed/main/blog_files/img/PicGo-Github/image-20210305153838568.png)

-   **Webserver 守护进程**：接受 HTTP 请求，通过 Python Flask Web 应用程序与 airflflow 进行交互。Webserver 提供功能的功能包括：中止、恢复、触发任务；监控正在运行的任务，断点续跑任务；查询任务的状态，日志等详细信息。

-   **Scheduler 守护进程**：<u>周期性地轮询</u>任务的调度计划，以确定是否触发任务执行。

-   **Worker 守护进程**：Worker负责启动机器上的executor来执行任务。使用celeryExecutor后可以在多个机器上部署worker服务。

### 2. 重要概念

**DAG（Directed Acyclic Graph）有向无环图**

-   在Airflflow中，一个DAG定义了一个完整的作业。同一个DAG中的所有Task拥有相同的调度时间。

-   参数：

    -   dag_id：唯一识别DAG

    -   default_args：默认参数，如果当前DAG实例的作业没有配置相应参数，则采用DAG实例的default_args中的相应参数

    -   schedule_interval：配置DAG的执行周期，可采用 crontab 语法

**Task**

-   Task为DAG中<u>具体的作业任务</u>，依赖于DAG，必须存在于某个DAG中。Task在DAG中可以配置依赖关系

-   参数：

    -   dag：当前作业属于相应DAG

    -   task_id：任务标识符

    -   owner：任务的拥有者

    -   start_date：任务的开始时间

## （二） Airflow 安装部署

### 1. 安装依赖

-   CentOS 7.X

-   Python 3.5或以上版本（推荐）

-   MySQL 5.7.x

-   Apache-Airflflow 1.10.11

-   保证虚拟机能上网，需在线安装包

**注：正式安装之前给虚拟机做一个备份；**

### 2. Python 环境准备

备注：

-   提前下载 Python-3.6.6.tgz

-   使用 linux122 安装

```shell
# 卸载 
mariadb rpm -qa | grep mariadb 
mariadb-libs-5.5.65-1.el7.x86_64 
mariadb-5.5.65-1.el7.x86_64
mariadb-devel-5.5.65-1.el7.x86_64

yum remove mariadb 
yum remove mariadb-libs

# 安装依赖 
rpm -ivh mysql57-community-release-el7-11.noarch.rpm

yum install readline readline-devel -y
yum install gcc -y 
yum install zlib* -y 
yum install openssl openssl-devel -y 
yum install sqlite-devel -y 
yum install python-devel mysql-devel -y

# 提前到python官网下载好包 
tar -zxvf Python-3.6.6.tgz

# 安装 python3 运行环境 
cd Python-3.6.6/

# configure文件是一个可执行的脚本文件。如果配置了--prefix，安装后的所有资源文件 都会放在目录中 
./configure --prefix=/usr/local/python3.6 
make && make install 
/usr/local/python3.6/bin/pip3 install virtualenv

# 启动 python3 环境 
cd /usr/local/python3.6/bin/ 
./virtualenv env 
. env/bin/activate 

# 检查 python 版本 
python -V
```

### 3. 安装 Airflow

```shell
# 设置目录（配置文件） 
# 添加到配置文件/etc/profile。未设置是缺省值为 ~/airflow 
export AIRFLOW_HOME=/opt/lagou/servers/airflow 

# 使用豆瓣源非常快。-i: 指定库的安装源（可选选项） 
pip install apache-airflow -i https://pypi.douban.com/simple 

# 备注：可以设置安装的版本【不用执行】 
pip install apache-airflow=1.10.10 -i https://pypi.douban.com/simple
```

备注：

-   软件安装路径在$AIRFLOW_HOME（缺省为~/airflflow），此时目录不存在

-   安装的是版本是1.10.11，不指定下载源时下载过程非常慢

### 4. 创建数据库用户并授权

```sql
-- 创建数据库
create database airflowlinux122;

-- 创建用户airflow，设置所有ip均可以访问 
create user 'airflow'@'%' identified by '12345678'; 
create user 'airflow'@'localhost' identified by '12345678'; 

-- 用户授权，为新建的airflow用户授予Airflow库的所有权限 
grant all on airflowlinux122.* to 'airflow'@'%'; 
SET GLOBAL explicit_defaults_for_timestamp = 1;
flush privileges;
```

### 5. 修改 Airfow DB 配置

```sh
# python3 环境中执行 
pip install mysqlclient 
airflow initdb
```

备注：有可能在安装完Airflflow找不到 `$AIRFLOW_HOME/airflflow.cfg` 文件，执行完airflflow initdb才会在对应的位置找到该文件。



**修改 $AIRFLOW_HOME/airflflow.cfg：**

```properties
# 约 75 行 
sql_alchemy_conn = mysql://airflow:12345678@linux123:3306/airflowlinux122 

# 重新执行 
airflow initdb
```



**可能出现的错误：**

-   Exception: Global variable explicit_defaults_for_timestamp needs to be on (1) for mysql

**解决方法：**

```sql
SET GLOBAL explicit_defaults_for_timestamp = 1; FLUSH PRIVILEGES;
```

### 6. 安装密码模块

**安装 password组件：**

-   `pip install apache-airflow[password]`

**修改 airflflow.cfg 配置文件（第一行修改，第二行增加）：**

```properties
# 约 281 行 
[webserver] 

# 约 353行 
authenticate = True 
auth_backend = airflow.contrib.auth.backends.password_auth
```

**添加密码文件**

-   python命令，执行一遍；添加用户登录，设置口令
-   <u>注：启动 python shell 窗口</u>

```python
import airflow 
from airflow import models, settings 
from airflow.contrib.auth.backends.password_auth import PasswordUser 

user = PasswordUser(models.User()) 
user.username = 'airflow' 
user.email = 'airflow@lagou.com' 
user.password = 'airflow123' 
session = settings.Session() 
session.add(user) 
session.commit() 
session.close() 
exit()
```

### 7. 启动服务

```shell
# 备注：要先进入python3的运行环境 
cd /usr/local/python3.6/bin/ 
./virtualenv env 
. env/bin/activate 

# 退出虚拟环境命令 
deactivate 

# 启动scheduler调度器: 
airflow scheduler -D 

# 服务页面启动： 
airflow webserver -D
```

备注：

-   airflflow命令所在位置：`/usr/local/python3.6/bin/env/bin/airflflow `

-   安装完成，可以使用浏览器登录 linux122:8080；输入用户名、口令：airflflow 、airflflow123

### 8. 修改时区

Airflflow 默认使用 UTC 时间，在中国时区需要用+8小时。将UTC修改为中国时区，需要修改 Airflflow 源码

**1、修改 $AIRFLOW_HOME/airflflow.cfg 文件**

```properties
# 约 65 行 
default_timezone = Asia/Shanghai
```

**2、修改 timezone.py** 

```sh
# 进入Airflow包的安装位置 
cd /usr/local/python3.6/bin/env/lib/python3.6/site-packages/ 

# 修改airflow/utils/timezone.py 
cd airflow/utils 
vi timezone.py
```

第27行注释，增加29-37行：

```python
27 # utc = pendulum.timezone('UTC') 
28
29 from airflow import configuration as conf 
30 try: 
31 		tz = conf.get("core", "default_timezone") 
32 		if tz == "system": 
33 			utc = pendulum.local_timezone() 
34 		else: 
35 			utc = pendulum.timezone(tz) 
36 except Exception: 
37 		pass
```

**备注：以上的修改方式有警告，可以使用下面的方式（==推荐==）：**

```python
27 # utc = pendulum.timezone('UTC') 
28
29 from airflow import configuration
30 try: 
31 		tz = configuration.conf("core", "default_timezone") 
32 		if tz == "system": 
33 			utc = pendulum.local_timezone() 
34 		else: 
35 			utc = pendulum.timezone(tz) 
36 except Exception: 
37 		pass
```

<u>修改utcnow()函数 (注释掉72行，增加73行内容)：</u>

```python
62 def utcnow(): 
63 """ 
64 		Get the current date and time in UTC 
65
66 		:return:
67 """ 
68
69 # pendulum utcnow() is not used as that sets a TimezoneInfo object
70 # instead of a Timezone. This is not pickable and also creates issues
71 # when using replace() 
72 # d = dt.datetime.utcnow() 
73 d = dt.datetime.now() 
74 d = d.replace(tzinfo=utc) 
75
76 return d
```

**3、修改airflflow/utils/sqlalchemy.py**

```shell
# 进入Airflow包的安装位置 
cd /usr/local/python3.6/bin/env/lib/python3.6/site-packages/ 

# 修改 airflow/utils/sqlalchemy.py 
cd airflow/utils 
vi sqlalchemy.py
```

在38行之后增加 39 - 47 行的内容：

```python
38 utc = pendulum.timezone('UTC') 
39 from airflow import configuration as conf 
40 try: 
41 		tz = conf.get("core", "default_timezone") 
42 		if tz == "system": 
43 			utc = pendulum.local_timezone()
44 		else: 
45 			utc = pendulum.timezone(tz) 
46 except Exception: 
47 		pass
```

**备注：以上的修改方式有警告，可以使用下面的方式（推荐）：**

```python
38 utc = pendulum.timezone('UTC') 
39 from airflow import configuration 
40 try: 
41 		tz = configuration.conf("core", "default_timezone") 
42 		if tz == "system": 
43 			utc = pendulum.local_timezone()
44 		else: 
45 			utc = pendulum.timezone(tz) 
46 except Exception: 
47 		pass
```

**4、修改airflflow/www/templates/admin/master.html**

```shell
# 进入Airflow包的安装位置 
cd /usr/local/python3.6/bin/env/lib/python3.6/site-packages/ 

# 修改 airflow/www/templates/admin/master.html 
cd airflow/www/templates/admin 
vi master.html
```

```python
# 将第40行修改为以下内容： 
40 		var UTCseconds = x.getTime(); 

# 将第43行修改为以下内容： 
43 		"timeFormat":"H:i:s",
```



**重启airflflow webserver：**

```shell
# 关闭 airflow webserver 对应的服务 
ps -ef | grep 'airflow-webserver' | grep -v 'grep' | awk '{print $2}' | xargs -i kill -9 {} 

# 关闭 airflow scheduler 对应的服务 
ps -ef | grep 'airflow' | grep 'scheduler' | awk '{print $2}' | xargs -i kill -9 {} 

# 删除对应的pid文件 
cd $AIRFLOW_HOME 
rm -rf *.pid 

# 重启服务（在python3.6虚拟环境中执行） 
airflow scheduler -D 
airflow webserver -D
```

### 9. Airflflow 的 web界面

![image-20210305161826479](https://raw.githubusercontent.com/LanceMai/MyPictureBed/main/blog_files/img/PicGo-Github/image-20210305161826479.png)

-   Trigger Dag：人为执行触发

-   Tree View：当dag执行的时候，可以点入，查看每个task的执行状态（基于树状视图）。状态：success、running、failed、skipped、retry、queued、no status

-   Graph View：基于图视图（有向无环图），查看每个task的执行状态

-   Tasks Duration：每个task的执行时间统计，可以选择最近多少次执行

-   Task Tries：每个task的重试次数

-   Gantt View：基于甘特图的视图，每个task的执行状态

-   Code View：查看任务执行代码

-   Logs：查看执行日志，比如失败原因

-   Refresh：刷新dag任务

-   Delete Dag：删除该dag任务

### 10. 禁言自带的 DAG 任务

停止服务：

```shell
# 关闭 airflow webserver 对应的服务 
ps -ef | grep 'airflow-webserver' | grep -v 'grep' | awk '{print $2}' | xargs -i kill -9 {} 

# 关闭 airflow scheduler 对应的服务 
ps -ef | grep 'airflow' | grep 'scheduler' | awk '{print $2}' | xargs -i kill -9 {} 

# 删除对应的pid文件 
cd $AIRFLOW_HOME 
rm -rf *.pid
```

修改文件 $AIRFLOW_HOME/airflflow.cfg：

```properties
# 修改文件第 136 行 
136 # load_examples = True 
137 load_examples = False 

# 重新设置db 
airflow resetdb -y
```

重新设置账户、口令：

```python
import airflow 
from airflow import models, settings 
from airflow.contrib.auth.backends.password_auth import PasswordUser 

user = PasswordUser(models.User()) 
user.username = 'airflow' 
user.email = 'airflow@lagou.com' 
user.password = 'airflow123' 

session = settings.Session() 
session.add(user) 
session.commit() 
session.close() 
exit()
```

重启服务:

```shell
# 重启服务 
airflow scheduler -D 
airflow webserver -D
```

### 11. crontab 简介

-   Linux 系统则是由 cron (crond) 这个系统服务来控制的。Linux 系统上面原本就有非常多的计划性工作，因此这个系统服务是默认启动的

-   Linux 系统也提供了Linux用户控制计划任务的命令：**crontab 命令**

![image-20210305162511725](https://raw.githubusercontent.com/LanceMai/MyPictureBed/main/blog_files/img/PicGo-Github/image-20210305162511725.png)



-   日志文件：ll /var/log/cron*

-   编辑文件：vim /etc/crontab

-   进程：ps -ef | grep crond ==> /etc/init.d/crond restart

-   作用：任务（命令）定时调度（如：定时备份，实时备份）

-   简要说明：cat /etc/crontab

---



## （三）任务集成部署

### 1. Airflow 核心概念

-   **DAGs**：有向无环图(Directed Acyclic Graph)，将所有需要运行的tasks按照依赖关系组织起来，描述的是所有tasks执行的顺序；

-   **Operators**：Airflflow内置了很多operators

    -   BashOperator 执行一个bash 命令
    -   PythonOperator 调用任意的 Python 函数

    -   EmailOperator 用于发送邮件

    -   HTTPOperator 用于发送HTTP请求

    -   SqlOperator 用于执行SQL命令

    -   自定义 Operator

-   **Tasks**：Task 是 Operator 的一个实例；

-   **Task Instance**：由于Task会被重复调度，每次task的运行就是不同的 Task instance。Task instance 有自己的状态，包括 success 、running 、failed 、skipped 、up_for_reschedule 、 up_for_retry 、queued 、no_status等；

-   **Task Relationships**：DAGs 中的不同 Tasks 之间可以有依赖关系；

-   **Executor**：Airflflow支持的执行器就有四种：

    -   <u>SequentialExecutor</u>：单进程顺序执行任务，默认执行器，通常只用于测试

    -   <u>LocalExecutor</u>：多进程本地执行任务

    -   <u>CeleryExecutor</u>：分布式调度，生产常用。Celery是一个分布式调度框架，其本身无队列功能，需要使用第三方组件，如RabbitMQ
    -   <u>DaskExecutor</u> ：动态任务调度，主要用于数据分析
    -   执行器的修改。修改 $AIRFLOW_HOME/airflflow.cfg 第 70行： executor = LocalExecutor 。修改后启动服务

### 2. 入门案例

放置在 $AIRFLOW_HOME/dags 目录下：

```python
from datetime import datetime, timedelta 

from airflow import DAG 
from airflow.utils import dates
from airflow.utils.helpers import chain
from airflow.operators.bash_operator import BashOperator 
from airflow.operators.python_operator import PythonOperator 

def default_options(): 
    default_args = { 
        'owner':'airflow', 					# 拥有者名称 
        'start_date': dates.days_ago(1), 	# 第一次开始执行的时间 
        'retries': 1, 						# 失败重试次数 
        'retry_delay': timedelta(seconds=5) # 失败重试间隔 
    }
    return default_args 

# 定义DAG 
def task1(dag):
    t = "pwd" 					# operator支持多种类型，这里使用 BashOperator
    task = BashOperator( 
        task_id='MyTask1', 		# task_id 
        bash_command=t, 		# 指定要执行的命令 
        dag=dag 				# 指定归属的dag 
    )
    return task 

def hello_world(): 
    current_time = str(datetime.today()) 
    print('hello world at {}'.format(current_time)) 
    
def task2(dag): 
    # Python Operator 
    task = PythonOperator( 
        task_id='MyTask2', 
        python_callable=hello_world, 	# 指定要执行的函数 
        dag=dag) 
    return task 

def task3(dag): 
    t = "date" 
    task = BashOperator( 
        task_id='MyTask3', 
        bash_command=t, 
        dag=dag) 
    return task 

with DAG(
    'HelloWorldDag', 					# dag_id 
    default_args=default_options(), 	# 指定默认参数 
    schedule_interval="*/2 * * * *" 	# 执行周期，每分钟2次 
) as d: 
    task1 = task1(d) 
    task2 = task2(d) 
    task3 = task3(d) 
    chain(task1, task2, task3) 			# 指定执行顺序
```

执行命令：

```shell
# 执行命令检查脚本是否有错误。如果命令行没有报错，就表示没问题 
python $AIRFLOW_HOME/dags/helloworld.py 

# 查看生效的 dags 
airflow list_dags -sd $AIRFLOW_HOME/dags 

# 查看指定dag中的task 
airflow list_tasks HelloWorldDag 

# 测试dag中的task 
airflow test HelloWorldDag MyTask2 20200801
```

### 3. 核心交易调度任务集成

**核心交易分析**

```shell
# 加载ODS数据（DataX迁移数据） 
/data/lagoudw/script/trade/ods_load_trade.sh 

# 加载DIM层数据 
/data/lagoudw/script/trade/dim_load_product_cat.sh /data/lagoudw/script/trade/dim_load_shop_org.sh
/data/lagoudw/script/trade/dim_load_payment.sh
/data/lagoudw/script/trade/dim_load_product_info.sh

# 加载DWD层数据 
/data/lagoudw/script/trade/dwd_load_trade_orders.sh 

# 加载DWS层数据 
/data/lagoudw/script/trade/dws_load_trade_orders.sh 

# 加载ADS层数据 
/data/lagoudw/script/trade/ads_load_trade_order_analysis.sh
```



备注： depends_on_past ，设置为True时，上一次调度成功了，才可以触发。

`$AIRFLOW_HOME/dags`

```python
from datetime import timedelta 
import datetime 
from airflow import DAG 
from airflow.operators.bash_operator import BashOperator 
from airflow.utils.dates import days_ago

# 定义dag的缺省参数
default_args = { 
    'owner': 'airflow',
    'depends_on_past': False, 
    'start_date': '2020-06-20', 
    'email': ['airflow@example.com'],
    'email_on_failure': False, 
    'email_on_retry': False,
    'retries': 1, 
    'retry_delay': timedelta(minutes=5), 
}

# 定义DAG 
coretradedag = DAG( 
    'coretrade', 
    default_args=default_args,
    description='core trade analyze', 
    schedule_interval='30 0 * * *', 
)

today=datetime.date.today() 
oneday=timedelta(days=1) 
yesterday=(today-oneday).strftime("%Y-%m-%d") 

odstask = BashOperator( 
    task_id='ods_load_data', 
    depends_on_past=False,
    bash_command='sh /data/lagoudw/script/trade/ods_load_trade.sh ' + yesterday, 
    dag=coretradedag 
)

dimtask1 = BashOperator( 
    task_id='dimtask_product_cat', 
    depends_on_past=False, 
    bash_command='sh /data/lagoudw/script/trade/dim_load_product_cat.sh ' + yesterday, dag=coretradedag 
)

dimtask2 = BashOperator(
    task_id='dimtask_shop_org', 
    depends_on_past=False, 
    bash_command='sh /data/lagoudw/script/trade/dim_load_shop_org.sh ' + yesterday, dag=coretradedag 
)

dimtask3 = BashOperator( 
    task_id='dimtask_payment', 
    depends_on_past=False,
    bash_command='sh /data/lagoudw/script/trade/dim_load_payment.sh ' + yesterday, dag=coretradedag 
)

dimtask4 = BashOperator( 
    task_id='dimtask_product_info', 
    depends_on_past=False, 
    bash_command='sh /data/lagoudw/script/trade/dim_load_product_info.sh ' + yesterday, 
    dag=coretradedag 
)

dwdtask = BashOperator( 
    task_id='dwd_load_data', 
    depends_on_past=False,
    bash_command='sh /data/lagoudw/script/trade/dwd_load_trade_orders.sh '+ yesterday, 
    dag=coretradedag 
)

dwstask = BashOperator( 
    task_id='dws_load_data', 
    depends_on_past=False, 
    bash_command='sh /data/lagoudw/script/trade/dws_load_trade_orders.sh ' + yesterday, 
    dag=coretradedag
)

adstask = BashOperator(
    task_id='ads_load_data', 
    depends_on_past=False, 
    bash_command='sh /data/lagoudw/script/trade/ads_load_trade_order_analysis.sh ' + yesterday, 
    dag=coretradedag 
)

odstask >> dimtask1 
odstask >> dimtask2
odstask >> dimtask3 
odstask >> dimtask4 
odstask >> dwdtask

dimtask1 >> dwstask 
dimtask2 >> dwstask 
dimtask3 >> dwstask 
dimtask4 >> dwstask 
dwdtask >> dwstask 

dwstask >> adstask
```

airflflow list_dags

airflflow list_tasks coretrade --tree

![image-20210305164428989](https://raw.githubusercontent.com/LanceMai/MyPictureBed/main/blog_files/img/PicGo-Github/image-20210305164428989.png)