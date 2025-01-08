# Pip install 报错：SSLError

## 环境

系统： macOS M3

python 版本：3.12.4

pip 版本：24.3.1

## 使用 pip 安装包

```shell
pip install torch torchvision torchaudio
```

报错:

```shell
WARNING: Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1000)'))': /pypi
WARNING: Retrying (Retry(total=3, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1000)'))': /pypi
```

这个报错主要是pip 升级后，源的证书不信任。

## 解决

### 临时方案

在 `pip install` 后面添加 `--trusted-host pypi.org --trusted-host files.pythonhosted.org`

```shell
pip install torch torchvision torchaudio --trusted-host pypi.org --trusted-host files.pythonhosted.org
```

### 永久方案

修改 `pip.conf` 文件，如果不存在，则新增

**Mac OS**

```shell
pip config list -v

For variant 'global', will try loading '/Library/Application Support/pip/pip.conf'
For variant 'user', will try loading '/Users/<username>/.pip/pip.conf'
For variant 'user', will try loading '/Users/<username>/.config/pip/pip.conf'
For variant 'site', will try loading '/Users/<username>/python/AI/RAG/.venv/pip.conf'
```

按需修改，改 global就是全局

```shell
vim /Users/silent.mo/.pip/pip.conf

[global]
trusted-host = pypi.python.org
               pypi.org
               files.pythonhosted.org
disable-pip-version-check = true
```

