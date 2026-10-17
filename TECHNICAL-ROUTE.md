# 技术路线：Bonsai × NInfer × KVMem

## 目标和边界

把 27B 三值权重、单 GPU 原生推理和检索式长上下文接到同一条可运行路径，并向网页 Agent 提供可观测、可取消、可续接的本机 API。三个发布目标分别是 RTX 4060 正常权重、RTX 4060 消融权重和 RTX 5060；前两者只应更换权重及缓存身份，后者必须使用对应架构引擎。

本页区分实现机制、运行配置和验收证据，不是完整包已发布、所有功能已启用或特定硬件已跑通的证明。

## 1. 权重与 artifact

源模型使用 PTQ1_0 三值表示；Native 消费 v3 `.ninfer`，不是在运行时用 GGUF 模拟另一套后端。转换器负责物理对象和逻辑绑定；loader 按实际组件绑定 Text、Vision 和 MTP。

消融变体从固定公开版本移入 98 个 writer 矩阵，保持格式和形状。其余 965 个原生对象逐字节保留；artifact ID 和缓存目录同时隔离，不能续接另一套权重计算出的 KV。

源码入口：`tools/artifact/reader.py`、`src/models/qwen3_5/load/mtp.cpp`。

## 2. 不同的 KV 页面合同

Native 以 64-token Device 页组织 KV，KVMem 本地合同使用 128-token 块/槽。简单把页号当成同一索引会混淆布局、偏移和回收所有权。

`KVMemPageBackend` 实现 `kvmem::KvMemBackend`，把一个 KVMem 块映射为两个 Native 页，沿用 Native Host KV 布局归档和恢复。先预留槽身份，再直接接管模型真正写出的页，减少同一批 token 的重复 GPU 占位分配；同时检查部分尾页、copy-on-write、引用计数、截断和 execution-address 生命周期。

代码：`src/models/qwen3_5/program/storage/kvmem_page_backend.h`、`src/models/qwen3_5/program/storage/kvmem_page_backend.cpp`。减少这条路径的重复页分配，不等于整个进程内存减少某个固定百分比。

## 3. CPU 历史与 GPU 工作集

完整档目标为 128K 输入、32K 检索历史、16K 生成预留。CPU 保留完整逻辑历史，GPU 安装选中的工作集和未来生成空间；逻辑位置不因检索重排而随意改变。

Host 存储按 Runtime 私有、按需增长，避免每个空会话先占满最大 CPU KV。修改保留在 `third_party/kvmem/kvmem/include/kvmem/kvmem_backend.hpp` 和 `third_party/kvmem/kvmem/src/host/kvmem_runtime.cpp`。

单并发约束的是同时执行的请求，不代表只能保留一个历史会话。会话缓存 RAM 限额是软目标；GDN 状态、workspace、映射权重、恢复峰值和系统提交额度必须另算。页锁定存储也不能任意当作可换页内存。

## 4. 混合注意力与恢复

满注意力 KV 与线性注意力/GDN 循环状态不是同一种数据。检索到一组 KV 不足以恢复执行，还需要与精确 token/位置/媒体身份匹配的循环边界状态。

会话、请求规划和前缀身份层检查这些边界；取消、工具返回或重写后，只复用有证明的状态。任意未保存的最长公共前缀不自动成为恢复点。磁盘会话缓存也不等于可跨重启复用的持久聊天记忆。

入口：`src/models/qwen3_5/program/planning/request_plan.cpp`、`src/models/qwen3_5/program/prefix_identity.cpp`、`src/models/qwen3_5/program/storage/kvmem_conversation.cpp`。

## 5. 推测解码与 CUDA Graph

MTP 提案、目标验证和接受提交在同一个 Native Program 内完成。必须检查 `--spec mtp --draft-tokens N` 和运行计数，不能因为文件名含 MTP 就声称加速开启。Graph 同样需要核对，不能把关闭 Graph 的小型档称为同参数完整档。

MTP、DFlash、DFlash2 是互斥后端；应选与 artifact 组件匹配的一条执行路径。参见 `serving.md` 和 `src/product/speculative_options.h`。提案接受率取决于目标权重和任务，其他运行时的 MTP 增益不能替代本项目实测。

## 6. Ada 与 Blackwell

SM89 / Ada 和 SM120 / Blackwell 的工具链、cubins、内核支持和调度分别处理。旧包在 target finalize 阶段拒绝不匹配架构，不是删掉启动脚本的架构检查就能修好。

本地适配包括独立构建、GDN cooperative occupancy、注意力 workspace/SM 调度及 KV codec 的架构路径。SM120 产物没有可依赖的 PTX JIT 保底，必须运行对应二进制并在真正的 RTX 5060 验证。

入口：`src/core/device.cu`、`src/models/qwen3_5/program/planning/startup.cpp`。构建与静态 ELF 检查不等于目标硬件运行成功。

## 7. API、工具与观测

服务端将协议翻译为共同的生成合同。输出预留、工具定义/结果、thinking、流终止和取消必须在各层一致。

实际故障包括引擎预留 16384、客户端却显式请求 32768，以及旧网页连接 8097、当前 Native 在 18097。只改服务端默认值不能覆盖客户端输出额度；网页能打开也不等于模型可用或工具任务成功。

观测入口有 `/health`、`/v1/models`、`/metrics`、`/monitor`、`/monitor/stats`。TPS 应交代任务、缓存、thinking、MTP/Graph、温度与计时范围；包含工具等待的 Agent 总耗时不是纯解码 TPS。

## 8. 发布与验证

- SM89 正常/消融：同引擎、同标准参数，只更换权重、来源和缓存身份。
- SM120：独立引擎，完整档与手动小型回退档明确标注，不静默降级。
- Git 源码排除权重、聊天、凭据、缓存、个人配置和原始日志。
- 离线包在 Releases 提供文件清单、校验和、技术配置和验证状态；未上传的文件不写成可下载。
- 编译、脚本检查、短回复、真实工具任务、长上下文质量和目标硬件验收分别记账。

详见 `VALIDATION.md` 和 `PROVENANCE.md`。
