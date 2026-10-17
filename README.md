# Bonsai Native + KVMem · Windows / 8GB GPU

**Bonsai 27B 三值模型 × NInfer 原生推理 × KVMem 长上下文**，面向 Windows 低显存机器的原生页面集成与部署。

## 已有成绩

**128K 上下文配置，约 55 TPS。** 这是项目方提供的历史实测反馈。

128K 指上下文配置，不自动等于该次请求实际填满 128K。硬件/功耗、实际输入长度、MTP/Graph、采样参数和计时口径的完整复现记录待补；不将此历史反馈写成三个包都已完成同条件硬件验收。

## 三种发布目标

| 版本 | 架构 | 区别 |
|---|---|---|
| RTX 4060 正常包 | Ada / SM89 | 原版 Bonsai 2 27B PTQ1_0，保留 Vision/MTP |
| RTX 4060 消融包 | Ada / SM89 | 与正常包同引擎、同容量、同运行参数，仅替换权重及缓存身份 |
| RTX 5060 包 | Blackwell / SM120 | 独立架构引擎，完整档和手动小型回退档明确区分 |

完整档统一：**128K 输入、32K GPU 检索历史、16K 输出、48K GPU KV 池、单并发、RK4V4-E8、MTP1、CUDA Graph、mapped embedding / Vision、按需 CPU 历史和会话停泊**。

目标表不等于三个完整离线包都已上传。实际资产与验证状态以 Releases 和 manifest 为准；不会把旧小型包改名为完整版。4060 是部署目标，维护机实际为 RTX 4070 Laptop / SM8.9；5060 仍需目标硬件验收。

## 技术路线

```text
原版 / 消融 PTQ1_0 权重 → v3 .ninfer artifact
                      ↓
           SM89 / SM120 架构专用 Native 引擎
                      ↓
      原生分页 KV / GDN 循环状态 / MTP / CUDA Graph
                      ↓
       128-token KVMem 块 ↔ 两个 64-token Native 页
                      ↓
       CPU 按需归档、历史检索、GPU 工作集与会话恢复
                      ↓
         OpenAI 兼容本机 API → 网页 DSH / 工具调用
```

详细说明：[TECHNICAL-ROUTE.md](TECHNICAL-ROUTE.md)。

核心工程点包括：

- **原生页桥接与直接接管**：复用 Native 页布局、地址和引用计数，不为已预填充的同一批 token 再分配一套 GPU 占位页；处理尾块、copy-on-write、截断和归档边界。
- **按需 Host KV**：每个 Runtime 私有存储，空会话不预分配最大 CPU KV 负载。它不代表没有 RAM、页锁定内存或系统提交成本。
- **混合状态一致性**：KV 检索与 GDN 状态恢复共同检查精确 token、逻辑位置和媒体身份，而不是只搬一份 KV 就宣布续接成功。
- **架构适配**：SM89 与 SM120 分别编译、封装和校验；涉及 GDN occupancy、注意力 workspace/SM 调度和 KV codec 路径，不通过删除架构检查冒充兼容。
- **工具与生命周期**：处理输出预留、真实工具请求/结果、取消、流结束和会话续接；不将“网页能打开”当作 Agent 任务通过。

## 源码与使用

首版以**完整源码归档**分发，仓库保留项目入口和在线技术说明：

- [下载 v0.1.0 完整源码](Bonsai-Native-KVMem-v0.1.0-source.zip)
- [来源与修改清单](source-snapshot.json)
- [源码归档校验和](SOURCE-SHA256.txt)

归档包含完整 CMake 项目、Native 源码、修改后的 KVMem core、构建说明、机制测试、第三方许可，以及三个目标包的共享配置和离线封装脚本；不包含大权重、个人聊天/工具凭据、编译产物或 CUDA 工具链。

解压后先阅读归档内 `README.md`、`docs/BUILD-WINDOWS.md` 和 `distribution/profiles.json`。构建为各架构分别进行；包含测试文件不表示本次已运行全部测试。

服务默认 API：`http://127.0.0.1:18097/v1`，模型 ID：`bonsai-native`。网页 DSH 默认端口 3080。客户端输出上限必须不超过 **16384**；服务端默认值不会覆盖客户端显式发送的 32768。

只监听本机；未经认证和网络安全配置不要暴露公网。不会自动关闭无关应用、修改页面文件、超频或更改显示设置。

## 消融权重与验证边界

消融来源：[BoldingBuilds PTQ1_0](https://huggingface.co/BoldingBuilds/Ternary-Bonsai-2-27B-Abliterated-PTQ1_0-GGUF)，固定版本 `b5f8b912289f873f421e7cc6389d1bdda22b3721`。本地移入 98 个 writer 矩阵，不重新量化；其余 965 个原生对象逐字节保留，包含 tokenizer、Vision 和 MTP。

作者模型卡提示 thinking 模式存在不收尾风险，并建议先关闭 thinking。提案接受率和 TPS 可能随权重、任务变化；不能保证“同配置”就是“同速度”。

SM89 原工作树构建、`--help`、静态 ELF 检查已完成；完整 MTP1/Graph 档、持续 TPS、全进程 RAM 峰值、128K 召回质量和目标 RTX 5060 实机验收必须分别记录。一次 87°C 热降速且 MTP 未开启时观察到的约 35 TPS，不是对历史 55 TPS 的同条件复现。

## 归属与许可

这是基于上游的集成与部署工作，不声称首次组合 NInfer、Bonsai 和 KVMem。

- Native 直接上游：[JGamboa/ninfer-4090-windows](https://github.com/JGamboa/ninfer-4090-windows)。
- KVMem core：[kvmem/kvmem-llama.cpp](https://github.com/kvmem/kvmem-llama.cpp)，固定版本 `95a2155b75f41eada1398c86b19dcf9703581b30`。
- 公开同类工作：[ninfer-fusion-kvmem](https://github.com/1314521gjy/ninfer-fusion-kvmem)、[KVMem 的 NInfer 迁移说明](https://github.com/kvmem/kvmem/blob/master/docs/ninfer-all-migration-plan.md)。

源码遵守 [LICENSE](LICENSE) 与归档内的 NOTICE/第三方许可；模型、DSH 和运行库分别遵守自身许可。上游其他 GPU 的成绩不属于本项目的 8GB 实测结果。
