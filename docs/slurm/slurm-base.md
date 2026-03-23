登录节点做了资源限制，若直接在登录节点配置conda环境等可能会提示内存不足等错误，更不要在登录节点跑计算任务，可使用salloc申请一个交互式作业进行环境配置，登录节点和计算节点环境共享。


## 1、slurm集群登录

每套slurm集群有自己的访问端口，在“工作台”-“资源总览”中通过点击如图所示图标可显示集群的IP和端口信息。

平台公共的Slurm集群的IP和端口是固定的不会变动。连接方式跟访问普通的远程Linux服务器一致，若用WSL或者Linux终端，使用ssh连接集群，username替换为自己的"u+学号"。如果经常使用，请自行配置ssh免密登录。在首次使用ssh登录之前，需要在平台右上角“个人中心”管理界面重置密码，并使用充值后的密码进行ssh连接。
```bash
    ssh -p xx username@xx.xx.xx.xx
```

### 1.1 登录节点和计算节点

登录节点所在的服务器无GPU卡，仅供用户登录提交作业使用，在登录节点运行nvidia-smi时无回显，即无法调用GPU资源。因登录节点有资源限制，部分操作容易失败尤其容易出现OOM的错误，环境配置、代码调试等请都在计算节点操作，养成通过hostname区分是在登录节点和计算节点的习惯。

- 登录节点：用于作业提交、状态查看、连接到计算节点，无GPU，登录节点的`hostname`是`workstation`，比如：
```
[utest@workstation ~]
```

- 计算节点：任务计算，有GPU，`hostname`是节点编号，比如gpu1：
```
[utest@gpu1 ~]
```


### 1.2 VSCode远程连接slurm登录节点

[VSCode  Remote Linux prerequisites](https://code.visualstudio.com/docs/remote/linux) ：从 VS Code 1.86（2024年初）及之后的版本开始，其内置的 vscode-server 要求libc.so.6中的 glibc ≥ 2.28，libstdc++中的GLIBCXX >= 3.4.25。slurm集群默认的glibc版本是2.17，导致vscode无法远程连接slurm。右下角会显示一个正在连接中的提示小窗，点击提示信息中高亮的 details可以看到详细的log。

VSCode远程连接slurm的方式如下：

- 方式一：降级使用的VSCode版本到1.85.2及以下，下载地址：https://code.visualstudio.com/updates/v1_85。本地装两个不同版本的VSCode，低版本的专门用于访问slurm。
- 方式二：patch VS Code Server，未测试过，仅做参考，欢迎配置通过的同学反馈。
  - 官方说明：https://code.visualstudio.com/docs/remote/faq#_can-i-run-vs-code-server-on-older-linux-distributions
  
  - 开源的方案：https://github.com/MikeWang000000/vscode-server-centos7
  
  - 知乎上手动安装的教程：https://zhuanlan.zhihu.com/p/1909663308669260068

- 方式三：客户端用新版本，如下手动替换 server端版本，测试失败，仍会自动下载与客户端对应版本的server。
	
  - https://github.com/microsoft/vscode/tags?after=1.86.2  查看具体的commit id，1.85.2对应`8b3775030ed1a69b13e4f4c628c612102e30a681`.

```bash
mkdir -p ~/.vscode-server/bin/8b3775030ed1a69b13e4f4c628c612102e30a681
wget -O ~/.vscode-server/bin/8b3775030ed1a69b13e4f4c628c612102e30a681/vscode-server.tar.gz \
  https://update.code.visualstudio.com/commit:8b3775030ed1a69b13e4f4c628c612102e30a681/server-linux-x64/stable

tar -xzf ~/.vscode-server/bin/8b3775030ed1a69b13e4f4c628c612102e30a681/vscode-server.tar.gz -C ~/.vscode-server/bin/8b3775030ed1a69b13e4f4c628c612102e30a681 --strip-components=1
  
```

### 1.3 VSCode远程连接slurm计算节点
若要使用VSCode远程连接slurm，因slurm登录节点做了资源限制，直接远程连接可能失败，先使用salloc申请计算节点，然后利用ProxyJump远程连接计算节点。具体操作参考： https://hpc.cloud/d/280-vscode/2 ,重点是下面的VSCode配置文件，注意用户名和gpu1根据实际情况修改。
``` bash
Host： slurm       # 登陆节点别名，可任意
  HostName xx.xx.xx.xx
  Port xx
  User u20250101   # 替换为自己的用户名
  ForwardAgent yes

Host compute       # 计算节点别名，可任意
	Hostname gpu1    # 计算节点的IP地址或hostname，要跟salloc分配到的节点一致，根据实际情况修改
	User u20250101   # 替换为自己的用户名
	ProxyJump slurm  # 上面配置的登录节点别名
```


## 2、Slurm常用命令

### 2.1 sinfo查看集群状态
- sinfo 查看节点与分区状态，可以看到gpu和gpu-a30两个分区，gpu分区3个A100节点，对应的node编号是gpu[1-3]，gpu-a30是2个A30节点，对应node编号是gpu[5-6]。TIMELIMIT是2天。
```bash
[utest@workstation ~]$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
gpu*         up 2-00:00:00      3    mix gpu[1-3]
gpu-a30      up 2-00:00:00      2   idle gpu[5-6]
```

加-N选项具体查看每个节点的状态
```bash
[utest@workstation ~]$ sinfo -N
NODELIST   NODES PARTITION STATE
gpu1           1      gpu* mix
gpu2           1      gpu* mix
gpu3           1      gpu* mix
gpu5           1   gpu-a30 idle
gpu6           1   gpu-a30 idle
```

查看指定的分区情况：sinfo --partition=cpu



- mix 是部分占用，可以分配新的作业
- idle 是空闲，完全没有占用
- alloc 节点满载，某项资源达到100%不可分配新作业

### 2.2 squeue查看作业信息

squeue 查看队列中的所有作业信息

- Name为interract的是交互式作业。
- ST是状态，PD即pending处于挂起状态，第一个作业是PD状态，看后面的原因是QOSMaxGRESPerUser触发了 QOS（服务质量）中定义的“单用户最大通用资源（GRES）限制。用户有其他作业在运行，暂时挂起，等运行中的作业结束了，该作业会符合要求就会被分配。如果用户提交的作业请求即使等运行中的作业结束了也无法满足则会直接被拒绝。
```bash
[utest@workstation ~]$ squeue
      JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
      23054       gpu exp24_mu u2023112 PD       0:00      1 (QOSMaxGRESPerUser)
      22916       gpu  dhr_gpu u2023311  R 1-16:12:14      1 gpu1
      22944       gpu  dhr_gpu u2024311  R 1-13:02:42      1 gpu2
      23048       gpu interact u2201106  R   11:40:41      1 gpu1
```
详细查看队列中的作业情况

- `squeue -l` ，long report 选项，查看更详细的信息
- `squeue  -o "%.18i %.10P %.8j %.12u %.2t %.10M %.6D %.20b  %.20S %R" `，指定查看的内容和输出格式
	- i 是id，P是partion，j是job name，u是user， t是Time，S是Start time。
- `squeue --help`  查看帮助选项

### 2.3 QoS限制策略查看
QOS: Quality of Service，Slurm 用来控制资源配额、优先级和限制的策略集合。常用的限制类别如下：

- 作业数量类

	MaxJobs：最多同时运行的作业数
	
  MaxSubmitJobs：最多可提交的作业数（排队+运行）

- CPU 核心数
  
	MaxCPUs：整个 QOS 可用 CPU 总数

	MaxCPUsPerJob：单个作业最多 CPU 数

- 内存
	
  MaxMemPerNode：每节点最大内存
	
  MaxMemPerCPU：每核最大内存

- GPU（GRES）
	
  Slurm 用 GRES 表示 GPU，实际写法类似 gres/gpu=4
  
  MaxGRES：QOS 总 GPU 限制
	
  MaxGRESPerJob：每个作业最大 GPU 数
	
- Priority：QOS 优先级，部分用户会配置多个
- 时间限制
	
  - MaxWall / MaxWallDurationPerJob：最长运行时间

- 使用量限制
  
    GrpCPUs：整个账户/组最多用多少 CPU
    
    GrpMem：总内存限制
    
    GrpGRES：总 GPU 限制
    
    GrpJobs：总作业数

普通用户查看Qos限制方式：

- `scontrol show partition`查看分区的配置情况，包含部分限制
- `sacctmgr show qos format=Name,Priority,MaxWall,MaxTRESPU,MaxJobsPu,MaxSubmitPu` 空值表示没有限制
- `sacctmgr list qos` 


日常使用超算资源只需掌握简单的几条命令即可，具体详细的说明请参考[SLURM官方文档](https://slurm.schedmd.com/documentation.html)。


## 3、作业提交

Slurm作业分为交互式和批量式两种。交互式作业通常用于环境调试，长期后台计算的任务则以作业脚本的方式进行提交。

- salloc：申请所需的计算资源（CPU、内存、GPU等），申请成功后获得一个在该节点上的 Shell。
- srun：跟要执行的命令，资源分配后直接运行对应的命令，常用语简单任务，比如`srun -p cpu -n 4 --pty /bin/bash` 启动远程主机bash终端。
- sbatch：跟要执行的脚本，资源分配后执行对应的脚本，适合复杂的任务。
- scancel 取消作业


### 3.1 交互作业

集群的计算节点默认不允许用户直接登录，需要分配到计算资源后才运行登录。在登录到集群后，使用`salloc`命令分配资源，然后再ssh到分配的节点上进行处理。具体是gpu1还是gpu2或者其他节点，要看salloc执行完之后的命令提示。
```bash
[utest@workstation ~]$ salloc
salloc: Pending job allocation 22465
salloc: job 22465 queued and waiting for resources
salloc: job 22465 has been allocated resources
salloc: Granted job allocation 22465
salloc: Waiting for resource configuration
salloc: Nodes gpu1 are ready for job
[utest@workstation ~]$ ssh gpu1
Warning: Permanently added 'gpu1' (ED25519) to the list of known hosts.
Last login: Thu Sep 18 15:02:06 2025 from 192.168.0.1
```


计算完成后，使用`exit`命令退出节点，注意需要`exit`两次，第一次`exit`是从计算节点退出到登录节点，第二次`exit`是释放所申请的资源。交互作业记得确认释放掉资源，否则会一直计费直到默认时间到。
```bash
(pytorch) [utest@gpu1 build]$ exit
logout
Connection to gpu1 closed.
(base) [utest@workstation ~]$ exit
exit
salloc: Relinquishing job allocation 22468
(base) [utest@workstation ~]$
```

<!--
![image3](media/job_image4.png)
![image4](media/job_image5.png)
![image6](media/job_image7.png)
-->

### 3.2 批量作业
如果环境配置好，要跑时间长的计算任务，将程序执行命令按规则写成作业脚本，并通过`sbatch`提交计算。如果队列中作业较多会进行排队，分配资源后开始计费并自动开始计算，计算结束后会自动释放资源停止计费，资源调度更高效且对用户更实惠。计算过程中无法与环境交互，可以通过日志查看作业运行状态。

一个简单的脚本示例如下：
```
    #! /bin/bash
    ### 表示这是一个bash脚本

    #SBATCH --job-name=JOBNAME
    ### 设置该作业的作业名

    #SBATCH --nodes=2
    ### 指定该作业需要2个节点数

    #SBATCH --ntasks-per-node=2
    ### 每个节点所运行的进程数为2

    #SBATCH --time=2:00:00
    ### 作业最大的运行时间，超过时间后作业资源会被SLURM回收

    #SBATCH --comment project_name
    ### 指定从哪个项目扣费。如果没有这条参数，则从个人账户扣费

    mpirun hostname
    ### 程序的执行命令
```

上述中`###`为注释行。第一行表示这是一个bash脚本，第4-17行以`#SBATCH`开头的命令表示这些是需要slurm系统处理的参数。

如下所示，通过`sbatch+作业脚本名`提交作业，系统会返回作业编号，通过`squeue`命令可以看到作业运行状态，等作业执行完成后，默认会把程序的输出放到`slurm-作业编号.out`的文件中，可通过该文件查看程序的输出。
```bash
[sso@n1 test]$ sbatch job.sh
Submitted batch job 193
[sso@n1 test]$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
               193       cpu  JOBNAME      sso  R       0:01      2 n[3-4]
[sso@n1 test]$ ls
job.sh
[sso@n1 test]$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
[sso@n1 test]$ ls
job.sh  slurm-193.out
[sso@n1 test]$ head slurm-193.out
n3
n3
n2
n2
```



### 3.3 指定GPU和分区

如果作业用到GPU，需在脚本中增加`--gres=gpu:<number of card>`参数。例如`#SBATCH --gres=gpu:2`指定2张GPU卡。以下为GPU作业的一个示例：
```
    #! /bin/bash
    ### 表示这是一个bash脚本

    #SBATCH --job-name=gpu-example
    ### 该作业的作业名

    #SBATCH --nodes=1
    ### 该作业需要1个节点

    #SBATCH --ntasks=16
    ### 该作业需要16个CPU

    #SBATCH --partition=gpu
    ### 该作业在gpu分区中提交

    #SBATCH --gres=gpu:1
    ### 申请1块GPU卡

    #SBATCH --comment project_name
    ### 指定从哪个项目扣费。如果没有这条参数，则从个人账户扣费

    source ~/.bashrc
    ### 初始化环境变量
    
    nvidia-smi
    ### 程序的执行命令，可以是其他脚本比如 python test.py
```

如要指定分区使用-p选项，默认提交到gpu分区，通过 `sbatch xx.sh -p gpu-a30`指定分区用A30的卡。