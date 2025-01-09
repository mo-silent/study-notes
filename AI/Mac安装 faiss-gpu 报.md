# Mac 安装 faiss-gpu 报错：Could not find a version that satisfies the requirement faiss (from versions: none)

## 环境

系统：Mac OS M3

python 版本：3.12.4

pip 版本：24.3.1



 ## 使用 pip 安装时：

```shell
pip install faiss-gpu
```

报错：

```shell
ERROR: Could not find a version that satisfies the requirement faiss (from versions: none)
ERROR: No matching distribution found for faiss
```

[Pypi上的Faiss](https://pypi.org/project/faiss-gpu/) 只是 macOS 和 Linux 预构建的二进制文件集合，仅使用下面python 版本：

```shell
Python :: 3.6
Python :: 3.7
Python :: 3.8
Python :: 3.9
Python :: 3.10
```

## Conda 安装 faiss-gpu 时

```shell
conda install -c pytorch faiss-gpu
```

报错：

```shell
PackagesNotFoundError: The following packages are 

  - faiss
```

因为 faiss-gpu 不支持 macOS

[Faiss Install.md](https://github.com/facebookresearch/faiss/blob/main/INSTALL.md)

- The CPU-only faiss-cpu conda package is currently available on Linux (x86-64 and aarch64), OSX (arm64 only), and Windows (x86-64)

- faiss-gpu, containing both CPU and GPU indices, is available on Linux (x86-64 only) for CUDA 11.4 and 12.1
- faiss-gpu-raft containing both CPU and GPU indices provided by NVIDIA RAFT, is available on Linux (x86-64 only) for CUDA 11.8 and 12.1.

## 解决

使用 Conda 安装 faiss-cpu 版本

```shell
conda install -c pytorch faiss-cpu
```

![conda-install-faiss-cpu](https://gallery-lsky.silentmo.cn/i_blog/2025/01//conda-install-faiss-cpu.png)
