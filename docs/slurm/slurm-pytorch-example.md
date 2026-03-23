## 1、slurm运行PyTorch任务


### 1.1 配置Conda环境

GPU集群已安装Anaconda，需要在自己的根目录下通过编辑`.bashrc`文件来配置Anaconda的路径。`.bashrc`用于存储并加载个人的终端配置和环境变量。bash在每次启动时都会加载/

将`Anaconda`的可执行文件路径添加到`~/.bashrc`文件中PATH环境变量

- 方式一：用vi编辑`~/.bashrc`，添加`export PATH=/opt/app/anaconda3/bin:$PATH`:
- 方式二：直接执行以下命令添加`echo 'export PATH=/opt/app/anaconda3/bin:$PATH' >> ~/.bashrc`

然后运行`source`命令使配置的环境变量生效：

    $ source ~/.bashrc

### 1.2 使用Conda安装PyTorch

因登录节点做了资源限制，conda的配置和软件安装请在计算节点操作，先用salloc申请资源，再连接计算节点。另外base环境下会将包安装在公共的`/opt/app/anaconda3/`目录下，该目录普通用户无权限会导致安装失败，请先创建独立环境并激活后再安装对应的包。

    $ conda install pytorch torchvision cudatoolkit=10.1 -c pytorch

等安装结束后即可使用PyTorch。登录节点和计算节点共享文件系统，个人环境配置对登录节点和计算节点同时生效。

### 1.3 提交任务

使用slurm提交需要运行的程序。以下是使用`sbatch`脚本方式提交的样例`test.sh`。

    #!/bin/csh
    #SBATCH --nodes=1    
    #SBATCH --ntasks-per-node=1
    #SBATCH --time=4:00:00
    python pytorch_test.py

`sbatch`用于配置作业参数，`--nodes`为使用的节点数，`--ntasks-per-node`为每个节点所运行的任务数。其他详细的参数配置参考`该文档 <job>`。

然后在终端执行`sbatch test.sh`，slurm提交作业后，会将日志输出到用户根目录下，文件名为`slurm-作业编号.out`。

如果需要多个节点提供计算，需要在代码中加入多机多卡的API。

    trainer = Trainer(gpus=8, num_nodes=4, distributed_backend='ddp')

详细信息参考官方API文档：

<https://pytorch-lightning.readthedocs.io/en/latest/slurm.html?highlight=slurm>

## 2、slurm运行jupyter任务

首先创建脚本：
```bash
vim jupyterLab.sh
```

加入以下内容：
```bash
#!/bin/bash

#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --time=1:00:00      # 1小时
##SBATCH --output=./jupyter.log   # 指定日志路径

# jupyter 端口
JUPYTER_PORT=8888

# 获取节点的IP地址和名称
NODE_IP=$(hostname -I | awk '{print $1}')
NODE_NAME=$(hostname)

# 写入日志提示ssh转发的连接方式
echo "ssh -L 8888:${NODE_NAME}:8888 yourname@xx.xx.xx.xx"
# 启动 Jupyter Lab
jupyter lab --no-browser --port=${JUPYTER_PORT} --ip=${NODE_IP}
```

- SBATCH 的选项请根据实际需求进行更改
- yourname替换为自己的账号名

提交作业：`sbatch jupyterLab.sh`。

查看作业状态：`squeue -u $USER`。

确定作业已经被启动之后,查看输出日志：`cat slurm-jobid.out`。确保jupyter正常启动。如果在日志提示端口 8888 已在使用中，请取消此作业。并修改作业脚本中的8888端口（1024 以上的任何端口）。

打开一个新的终端窗口，windows 通过 cmd (命令提示符) 执行 ，Linux 或 Mac 通过 terminal 执行。使用下面的命令使用ssh的转发功能，yourname替换为自己的账号，ssh需要一直保持连接，若ssh断掉需重连。

```bash
ssh -L 8888:gpu1:8888 yourname@xx.xx.xx.xx -p xx
```

然后通过本地电脑打开浏览器，输入slurm-jobid.out中最后一行的地址即可打开jupyter lab界面。，此处示例为：
```bash
http://127.0.0.1:8888/lab?token=69d84beb89b4a65d2e4f790dc9796607fc880a13fd5133af
```

如果要在GPU节点运行jupyter，需要指定GPU资源。
```bash
#SBATCH --gres=gpu:1
```
