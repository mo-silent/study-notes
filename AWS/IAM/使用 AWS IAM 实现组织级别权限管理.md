---
title: 使用 AWS IAM 实现组织级别管理和多账号管理
slug: use-iam-implement-org-multi-account
categories:
  - IAM
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: d3c86cc1-6d79-44c6-b3a9-b3b197f4d8de
  publish: true
---
<!-- [!TOP] -->

# 使用 AWS IAM 实现组织级别管理和多账号管理

## 介绍
> [!TIP]
>
> 本文不做 AWS 服务概念解释

在企业中，如何管理多个AWS账号的权限和访问控制是一项复杂但至关重要的任务。并且处于成本的考虑，企业可能会与 AWS Partner 签约代付，这时候会失去 AWS Organization 和 IAM Identity Center 这两项 AWS 的组织账号管理服务。因此本文探讨如何在不依赖AWS Organizations和Identity Center的情况下，仅使用 AWS IAM 服务来实现组织级别的权限管理和跨账号的账号切换。

**账号划分:** 
1. 账号 A - 管理账号, 所有用户都通过这个账号跳转到资源账号, 这个账号正常情况不应该存在资源。只用于管理
2. 账号 B - 资源账号, 资源账号会存在多个, 如: 开发账号, 测试账号, 生产账号等等

## 步骤

### Step 1: 在账号 A (管理账号) 创建 IAM User Group
> [!TIP]
>
> 这一步主要是在管理账号的逻辑管理, 并不会有实际意义。因为在资源账号下的 IAM Role 信任策略中配置的是用户, 不能基于用户组配置。
> 
> 用户组在 AWS 上也是一个逻辑概念, 并不是一个实体。
1. 登陆 [AWS IAM Console](https://us-east-1.console.aws.amazon.com/iam/home), 左侧栏点击 User groups, 然后右侧点击 Create user group
2. 在创建用户组页面, 输入用户组名称。如: devops-admin (运维管理组)
3. 在创建用户组页面的 Attach permissions policies - Optional 中选择 AdministratorAccess 权限后, 最下面点击 Create  user group
   > 除了管理组之外的其他用户组应该不附加任何权限, 这个账号只是给他们用于跳转到对应资源账号下

   ![create-user-group](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-user-group.png)
   

### Step 2: 在账号 A (管理账号) 创建 IAM User

1. 登陆 [AWS IAM Console](https://us-east-1.console.aws.amazon.com/iam/home), 左侧栏点击 Users, 然后右侧点击 Create User
2. 创建用户页面, 输入用户名。勾选 `Provide user access to the AWS Management Console - *optional*`, 就会出现密码选项, 按个人选择。
   ![create-user](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-user-1.png)
3. 在设置权限这一步, 选择对应的 group 就好了。
   > 如果这个用户有特别的权限需求, 可以在这里附加对应的策略。不过建议不要特殊化
4. 下一步点击 Create user
5. 在 Retrieve password 这一步, 你可以将包含用户登陆信息的 csv 文件下载到本地
   ![Retrieve password](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-user-4.png)
6. 点击弹出框的 View user, 在用户信息页面复制 ARN (记住这个 arn 下面步骤需要)
   ![copy arn](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-user-5.png)
7. (Optional) Enabled MFA, 在用户详情页, 点击 Enabled without MFA 绑定一下 MFA 认证 (推荐谷歌的)

### Step 3-1 (资源账号不存在跳转角色): 在账号 B (资源账号) 创建 IAM Role
> [!IMPORTANT]
>
> 这一步要记住账号 B (资源账号) 的账号 ID 和角色名

1. 先退出前面的账号, 用账号 B 的管理员账号登陆到  [AWS IAM Console](https://us-east-1.console.aws.amazon.com/iam/home)
2. 在 IAM 页面左侧栏点击 Roles, 然后右侧点击 Create Role
3. 在创建角色页面, `Select trusted entity` 选择 AWS acount, `An AWS account` 选择 Another account, 输入账号 A 的账号 ID (也可以随便输)。
   
   > [!TIP]
   >
   > Options 的 Require MFA 按实际选, 但是在企业中建议选上
   
   ![trust aws account](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-role-1.png)
4. 下一步 `Add permissions`, 按实际配置一个权限, 这里配置 AdministratorAccess
   ![Add permissions](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-role-2.png)
5. 下一步输入 Role name 和 Description, 点击创建
   ![edit trusted entities](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-role-3.png)
6. 在弹出框中, 点击 View role
7. 在角色详情页面, 点击 Trust relationships, 然后点击 Edit trust policy
   ![Role details](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-role-4.png)
8. 在 Edit trust policy 页面, 修改 `Principal` 的 `AWS` 值为 Step 2 创建的 IAM User ARN
   ![Edit trust policy](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-role-5.png)
9. 修改完成后, 页面下拉到最后点击 Update policy 即可

### Step 3-2 (资源账号存在跳转角色): 在账号 B (资源账号) 修改已有的 IAM Role
1. 先退出前面的账号, 用账号 B 的管理员账号登陆到  [AWS IAM Console](https://us-east-1.console.aws.amazon.com/iam/home)
2. 在 IAM 页面左侧栏点击 Roles, 找到对应的 IAM Role, 点击进入到角色详情页
3. 在角色详情页面, 点击 Trust relationships, 然后点击 Edit trust policy
   ![Role details](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-role-4.png)
4. 在 Edit trust policy 页面, 修改 `Principal` 的 `AWS` 值为 Step 2 创建的 IAM User ARN
   ![Edit trust policy](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-create-role-5.png)
5. 修改完成后, 页面下拉到最后点击 Update policy 即可

### Step 4: 通过账号 A (管理账号) 跳转到账号 B (资源账号) 下
1. 用户登陆到 AWS Console, 点击右上角, 点击 Switch role。
   ![switch role](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-switch-role-2.png)
   如果开启了 Multi-seesion support, 点击 Add session 旁边的下拉箭头找到 Switch role
   ![switch role-1](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-switch-role-1.png)
2. 在切换角色页面, 输入账号 B 的账号 ID 和角色名称, 还可以选一个自己用来标识这个资源账号的颜色(多个资源账号强烈推荐)
   ![switch role pages](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-switch-role-3.png)
3. 跳过去后, 右上角点击就可以看 Current session
   ![Current session](https://gallery-lsky.silentmo.cn/i_blog/2025/07/IAM-switch-role-4.png)
