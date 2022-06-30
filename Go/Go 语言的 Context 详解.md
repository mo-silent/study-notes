# GO 语言中 Context 包详解

> Author mogd 2022-06-28
> Update mogd 2022-06-28
> Adage `Rreality is merely an illusion, albeit a very persistent one.`

## 前言

不知道有没有小伙伴跟我一样，在学习 go 语言基础的时候，遇到需要 `context` 信息的，都是直接传入 `context.TODO()`，并没有深入研究 `Context`

让笔者深入了解到 `Context` 包是因为自己在写一个 K8S CMDB 平台时，想到需要对数据初始化（就是创建 admin 用户，和测试用户，以及分配对应的 Casbin 权限）

因为 user 模块参考了 [gin-vue-admin](https://github.com/flipped-aurora/gin-vue-admin.git)，并且在 `GVA` 中也有数据初始化部分

刚开始以为很简单，结果认真一看，发现代码中使用了 `Context`；好家伙，知识盲区了，前面学习的时候

埋头苦学往往只是知其形而不知其意，只有实际做项目才能够真正的掌握一门语言
> 笔者自己写的一个 K8S CMDB平台后端接口，[gin-kubernetes](https://gitee.com/MoGD/gin-kubernetes.git)
> 这个刚开始写，写的很简单，感兴趣的可以看看；也希望大佬们提提建议
> user 模块参考的 [gin-vue-admin](https://github.com/flipped-aurora/gin-vue-admin.git)

笔者也是最近才认真学习了 `Context` 包，