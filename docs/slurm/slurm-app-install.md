
- 若为商业软件，请自行获取软件使用权并安装；
- 若为常用开源软件，查看/opt/app目录下是否已安装；
- 若未安装
	- 请先考虑是否能用conda 安装；
	- 再考虑在自己家目录下使用源码安装，遇到问题，请将可复现的步骤发到用户交流qq群咨询；
	- 或者考虑容器安装的方法。


## 1、预装软件的使用

/usr/bin等$PATH变量中指定的路径下的工具都可直接使用，另安装了spack,anaconda等软件在/opt目录下。

为了避免不恰当编辑 bash 配置文件引起的环境变量冲突、混乱的问题，同时为了方便不同版本的管理，集群配置了 module 工具来调用系统预装的软件。只有系统预装的软件才可以使用 module 工具来加载。

module的常用命令：

- module avail 查看所有可用的模块
- module avail xx 查看指定的模块
- module load module_name 来加载所需模块

示例：默认的/usr/bin/gcc版本是4.85，想更换更高版本的gcc，可通过以下命令实现：

```bash
$ gcc -v
$ which gcc
$ module avail
$ module load gcc-10.1.0-gcc-4.8.5-2new4ox
$ gcc -v
$ which gcc
/opt/app/spack/opt/spack/linux-centos7-haswell/gcc-4.8.5/gcc-10.1.0-2new4oxsi6o5ejrrjxsjtvvxubujfyyk/bin/gcc
```

也可通过`module switch`切换。如果需要恢复之前的gcc，直接`module unload gcc-10.1.0-gcc-4.8.5-2new4ox`即可。

在slurm任务脚本中，加载系统预装软件环境的命令和运行系统预装软件的命令，不能在脚本的同一行里。加载命令的一行，必须在运行命令的一行的上方。

部分预装软件需运行相关脚本初始化，例如： 
- 通过 source /opt/app/anaconda3/bin/activate 命令来激活conda环境
- 通过 source /opt/intel/bin/compilervars.sh intel64 命令来激活intel的编译套件
```bash
$ source /opt/intel/bin/compilervars.sh intel64
$ which mpirun
/opt/intel/compilers_and_libraries_2020.1.217/linux/mpi/intel64/bin/mpirun
$ which icc
/opt/intel/compilers_and_libraries_2020.1.217/linux/bin/intel64/icc
```

module是Environment Modules 开源项目的命令，提供用户环境的动态修改，详细使用请参考：

- https://github.com/cea-hpc/modules
- https://modules.readthedocs.io/en/latest/INSTALL.html
- https://en.wikipedia.org/wiki/Environment_Modules_(software)

`cat ${MODULESHOME}` 查看moudles的安装和配置目录。



## 2、自行安装软件

### 2.1 Conda
集群支持也允许通过conda安装自己需要的软件包：

- 使用集群预安装的版本：将Anaconda的可执行文件路径添加到~/.bashrc文件中PATH环境变量，`echo 'export PATH=/opt/app/anaconda3/bin:$PATH' >> ~/.bashrc`。
- 自行安装指定版本的conda：
```bash
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm -rf ~/miniconda3/miniconda.sh
~/miniconda3/bin/conda init bash
source ~/.bashrc

```

### 2.2 Spack


[Spack](https://spack.readthedocs.io/en/latest/index.html)是专为大型超级计算中心设计的自动化包管理器，支持各种平台和环境中的多个版本和软件配置。slurm集群已经设置安装好了spack，安装目录在`/opt/app/spack`。

如果预安装的软件没有自己想要的，需要管理员通过spack安装。因权限控制，集群不允许用户使用系统spack安装。不过=可以自己的目录下安装spack再安装其他软件。Spack 是 “包管理器”，负责下载、编译、安装软件，Modules 是 “环境切换器”，负责激活/停用已经安装好的软件，通过修改环境变量实现。

[conda和spack的区别](https://stackoverflow.com/questions/77873047/what-are-the-key-differences-between-spack-and-conda-package-managers)：

- Conda 使用预编译的软件包，通常已经包含了其依赖项，可以直接安装使用。
- Spack 支持从源码编译软件包，并且可以根据用户的需求配置编译器、编译选项、MPI 实现等。
- Conda 支持在不同虚拟环境中安装同一个软件包的不同版本。通过环境隔离来管理多版本的依赖。
- Spack 内置支持多个版本、不同编译器选项和编译选项的安装。一个用户可以在同一系统上安装多种配置的相同软件包，而不需要手动创建多个环境。

conda对于深度学习等领域应用更广泛，更容易上手。但都用conda，会使得有很多重复的软件包，浪费存储空间。


## 3、slurm上源码编译：glibc
conda中预编译好的软件通常不是最新的，或者没有自己想要的版本，则可以通过直接从源码编译的方式安装。

以glibc 2.29为例，glibc是GNU发布的c运行库，是linux系统中最底层的api。依赖glibc系统组件多，一般使用系统默认的，无法在整体部署高版本glibc，如果需要高版本glibc，可通过源码自行安装。集群上glibc对应的文件是/lib64/libc.so.6，`strings /lib64/libc.so.6 | grep GLIBC_` 可以看到当前 glibc 提供的 ABI 版本，最高版本是GLIBC_2.17，向后兼容所有更低版本的 GLIBC 接口，使用`ldd --version` 会显示ldd (GNU libc) 2.17。

先配置好conda并激活指定的环境，默认的base环境会安装软件到公共目录，没有写权限会导致安装失败。

申请计算节点并连接到计算节点，后续所有操作在计算节点操作，登录节点会因资源限制部分操作会失败。
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

依赖升级：系统默认的make是3.82版本较低，编译glibc 2.29需要make和gmake 4.0及以上的版本，slurm上gmake链接到make。系统默认 gcc是 4.8 无法编译 glibc 2.29，需要切换到gcc 7.5版本，10.1的版本过高。
```bash
conda install make  # 如果报错，使用conda clean --all --yes 清理缓存再安装
module load gcc-7.5.0-gcc-4.8.5-of6wn6o
```

然后执行以下命令下载解压并环境探测和配置构建，--prefix指定的安装目录前缀不要与源文件相同，否则后面安装会因为文件覆盖报错。
```bash
mkdir -p ${HOME}/opt/app/glibc-2.29
wget http://ftp.gnu.org/gnu/glibc/glibc-2.29.tar.gz
tar -zxvf glibc-2.29.tar.gz && cd glibc-2.29
mkdir build && cd build
../configure --prefix=${HOME}/opt/app/glibc-2.29 --disable-sanity-checks MAKE=$(which make)
```

若configure操作出现如下的错误提示，
```bash
LD_LIBRARY_PATH shouldn't contain the current directory when
*** building glibc
```
使用`echo $LD_LIBRARY_PATH` 检查下路径是否包含了 . (当前目录) 或空条目（代表当前目录），通常是最后多了一个:号，将内容复制去掉最后的:并重新配置环境变量`$LD_LIBRARY_PATH`,示例如下。
```bash
export LD_LIBRARY_PATH=/opt/app/spack/opt/spack/linux-centos7-haswell/gcc-4.8.5/gcc-7.5.0-of6wn6ob65uvcmzh2ycienc5fikcmov2/lib64:/opt/app/spack/opt/spack/linux-centos7-haswell/gcc-4.8.5/gcc-7.5.0-of6wn6ob65uvcmzh2ycienc5fikcmov2/lib:/usr/local/nvidia/lib
```

再编译安装
```bash
make -j16       # make[1]: Leaving directory '/home/utest/opt/glibc-2.29' 表示正常结束
make install    # make[1]: Leaving directory '/home/utest/opt/glibc-2.29'表示正常结束

export PATH=${HOME}/opt/app/glibc-2.29/bin:$PATH
ldd --version
```