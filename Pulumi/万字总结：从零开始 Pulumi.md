# Before begin

## Install Pulumi

#### Package

https://www.pulumi.com/docs/install/versions/

#### command line

**window (all in PowerShell)**

1. install choco： https://chocolatey.org/install

   ```shell
   Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
   ```

2. install Pulumi

   ```shell
   choco install pulumi
   ```

3. When the installation completes, you can test it out by reading the current version:

   ```shell
   pulumi version
   ```

**Linux**

```shell
curl -fsSL https://get.pulumi.com | sh
```

**MacOS**

```shell
brew install pulumi/tap/pulumi
```

## Configure and login Pulumi

1. create a Pulumi cloud account: https://app.pulumi.com/

2. create a Personal access tokens:
   ![access-tokent](https://gallery-lsky.silentmo.cn/i_blog/2025/01//pulumi-cloud-access-token.png)

3. configure `PULUMI_ACCESS_TOKEN` environment variable

4. logging in Pulumi cloud

   ```shell
   >pulumi login
   Logging in using access token from PULUMI_ACCESS_TOKEN
   Logged in to pulumi.com as mo-silent (https://app.pulumi.com/mo-silent)
   ```



