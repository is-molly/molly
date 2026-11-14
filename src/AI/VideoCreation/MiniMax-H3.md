---
order: 1
date: 2026-11-13
---

> 基于 2026-11 的 antirez/h3.c 与 skaiy/wild-h3c-comfyui README、MiniMaxAI/MiniMax-H3 模型卡与官方 awesome-minimax-h3-integration 索引，以及 modelfit.io 汇总的社区实测数据整理。

# MiniMax H3 本地部署与视频生产

## 1. 概述

### 1.1 模型概况

MiniMax H3 是视频生成模型：输入一段提示词，可附带图片、视频、音频引用，输出一段 24 fps、带立体声的视频，原生时长 4 到 15 秒。开放权重的 H3-Base 是 768p 模型，短边默认 768 像素；官方托管工作流的 2K 输出依赖 H3-Regenerate-2K 模块，该模块未开源，本地权重止步 768p。

模型由四个组件构成：

| 组件 | 职责 | 规模 |
|---|---|---|
| 扩散 Transformer（DiT） | 去噪主体 | 33B 参数，BF16 约 61.7 GiB |
| 文本编码器 Qwen3-VL-32B | 编码提示词与参考图理解 | BF16 约 62.1 GiB |
| video VAE | latent 与像素互转 | 约 5B，9.7 GiB |
| audio VAE | 音频轨同理 | 0.56 GiB |

官方仓库提供两个变体，checkpoint 尺寸相同：

- **FL2VA**：文生视频加首帧/末帧锚定。接受零、一或两张图，零张即纯文生视频。只用关键帧训练，原始出片质量通常更好。
- **Ref2VA**：参考模式。最多接受 9 张图、3 段视频（各 2–15 秒）、3 段音频，合计 12 个文件。接受的参考类型多，但基础质量略低于 FL2VA，社区的 148 MiB Ref Patch 可以部分弥补。

参考素材在提示词里按序号引用：`<Picture 1>`、`<Video 1>`、`<Audio 1>`，对话用 `<d>` 标记。模型只认序号，文件名对它没有意义。

### 1.2 开放程度

可下载的部分是 H3-Base 权重、推理参考实现和两份提示词编写指南（Base 与 Reference）。官方没有提供训练器，Diffusers 集成仅推理；微调方法、加速 LoRA（Turbo、PDD、FastH3 等）和量化重打包全部来自社区。

托管版 H3 Max 家族（含 Turbo、Director 变体）闭源，只能通过 fal 和 ComfyUI Partner Nodes 访问。GGUF、NVFP4、INT8 等量化格式数量庞大，但对应工具链（ComfyUI 加速节点、sd.cpp）基本围绕 NVIDIA CUDA 构建。

一个值得提前知道的体量关系：官方仓库全量约 464 GiB、280 个文件——根目录有 Diffusers 格式的裸 `transformer/` 和 `transformer_ref/`，另含两套自包含的 `FL2VA/`、`Ref2VA/` 管线目录。h3.c 只读后两者中的一棵树，下载必须按子目录圈定范围，直接全量下载会浪费一倍以上磁盘。

### 1.3 Mac 上的路线

| 路线 | 输入格式 | Apple Silicon 上的状态 |
|---|---|---|
| ComfyUI 官方 H3 节点 | safetensors / GGUF | 面向 NVIDIA CUDA 开发，官方文档未提及 Apple Silicon。PyTorch MPS 路径单片一小时以上，实际不可用 |
| GGUF + stable-diffusion.cpp | GGUF（最小 DiT 3.78 GiB） | 能跑，但文本编码器必须留在 CPU（`--backend te=cpu`），无已发表的 Mac 速度数据 |
| MLX 系 | MLX 包 | mlx-serve 的 4/8-bit 包（FastH3 Omni 8 步，需 macOS 26.2+）；Argus 块流式（24 GB M4 Pro 有实测）；Draw Things 用 Swift/Metal 实现，跑 8/6/4-bit 转换权重 |
| **h3.c** | 官方 BF16，不量化 | antirez 的纯 C + Metal 实现，MIT。文生视频/音频、首尾帧、Ref2VA 引用端到端可用，Apple Silicon 上目前唯一的实用级原生引擎 |

本文的结论是 h3.c。两点背景说明：

其一，最初需求是「Q8 量化版」。Q8_0 GGUF（剪枝版 19.97 GiB）属于 ComfyUI / sd.cpp 生态，h3.c 不读任何量化文件，只吃原始 BF16 树。选 h3.c 等于放弃量化格式的容量优惠，改用内存策略换精度。

其二，MLX 系路线各有适配档位：4-bit 包在 128 GB M4 Max 上的实测峰值内存 66.5–76.6 GB，超出 64 GB 机器；Argus 流式在 24 GB M4 Pro 上一段 5 秒 1344×768 耗时 47 分 58 秒；Draw Things 有 M5 Ultra 的官方基准（1024×512、124 帧、20 步，201.87 秒），但同样依赖量化转换权重。h3.c 同时满足 64 GB 装得下、官方精度、持续开发三个条件。

### 1.4 许可与硬件预算

**许可。** 权重受 MiniMax H3 Community License 约束（2026-08-02 生效）。排除区域为美国、英国、欧盟、韩国，适用区域为全球其余地区，中国大陆在内。商用有两点要注意：年收入超过 2000 万美元的公司需另行取得 MiniMax 书面授权；ComfyUI 文档口径更宽——本地生成内容的商用需向 MiniMax 购买商业许可（由 Comfy 代售）。适用区域内的个人学习与非商业使用无需申请，本文流程止于个人使用。

**硬件预算**（64 GB 内存只能是 M1 Max——M1 Ultra 仅出现在 Mac Studio——统一内存带宽 400 GB/s）：

| 资源 | 预算 | 说明 |
|---|---|---|
| 内存 | DiT 全驻留约 36.5 GiB（512×512 实测口径） | `--ssd-streaming` 可压到约 2 GiB，代价是前向慢 26–84%；DiT、编码器、VAE 分阶段加载，不同时驻留；`--show` 预览额外常驻约 10 GiB，低内存场景不要开 |
| 磁盘 | FL2VA 一棵树 134 GiB，两棵 268 GiB | 只做文生视频先只下 FL2VA，预留约 160 GB 可用空间 |
| 带宽 | 400 GB/s | 去噪是带宽受限负载，这是 M1 Max 的主要瓶颈；已发表速度数据均来自 M5 级芯片，M1 的预期见 4.4 节 |

下载的圈定范围与国内网络镜像在 3.2 节展开。

### 1.5 阅读路线

全文四层。第 1、2 章是认知层：模型是什么、引擎如何工作。第 3、4 章是路线一（命令行）：编译、下权重、出片，走完即具备完整生成能力。第 5 章是路线二（ComfyUI）：在路线一之上补可视化编排与云端混合，需要时再读。第 6、7 章是进阶层：单段拼成片（6）、流程自动化（7），对两条路线通用，不依赖第 5 章。第 8 章集中算账。只走命令行的话，读完第 4 章即可出片。

## 2. 原理

### 2.1 生成管线

一次生成分四个阶段顺序执行，同一时刻只有一个阶段的模型驻留内存：

1. **提示词编码**。Qwen3-VL-32B 把提示词转成 conditioning；带参考素材时，图片与视频先经视觉塔进入编码器。这一阶段 h3.c 做了流式处理：预取环在 GPU 算当前层时由 8 个 I/O 线程填充后续层权重，环深 M3 及更老硬件 2 层、M5 为 3 层。
2. **latent 去噪**。DiT 在视频/音频联合 latent 上迭代扩散，算力与带宽的大头。默认 20 次去噪，使用官方发布的 video/audio 双轨 shifted schedule，Euler 积分，最后一次去噪后补一个终态零点。
3. **双 VAE 解码**。video VAE 将视频 latent 解为 RGB24 帧；音频轨由 AudioVAE 配 BigVGAN 解出 32 kHz 立体声。
4. **封装**。帧与 PCM 经两根并发管道直喂 ffmpeg，编码 H.264 + 32 kHz 立体声 AAC，中间不产生未压缩文件。

参考素材在第一阶段进入：首尾帧走 visual VAE 编码 + Qwen3-VL 视觉塔 + 三层 deepstack 呈现，条件增强系数 0.999，首帧拉伸到目标画幅、末帧按 cover 裁切；参考视频解码为 24 fps 后按因果 ceil(T/4) 压缩、两帧采样、以带时间戳的 `<Video N>` 呈现；参考音频转 32 kHz 立体声 F32 后取 AudioVAE 后验均值，按 0.999 干净 latent 加 0.001 种子噪声混合，钉在时间步 1.0。这些路径与官方参考实现对齐，h3.c 给出的对齐误差：音频编码器相对 L2 3.59e-6，端到端波形 6.94e-5。

模型在时间和空间上都是离散化的，两条硬规则直接约束后面的用法：

- **帧数向上对齐到 5 + 17n**。请求 23 帧会得到 39 帧，`--seconds 10` 实际生成 243 帧（10.125 秒）：

| 帧数 | 时长 |
|---|---|
| 22 | 0.917 s |
| 39 | 1.625 s |
| 56 | 2.333 s |
| 107 | 4.458 s |
| 243 | 10.125 s |
| 362 | 15.083 s |

- **宽高均为 32 的倍数，乘积不超过 768×1344**。768×1344 是横屏上限，竖屏对称；H3-Base 本身是 768p 模型，这是发布画幅，不是算力上限。

### 2.2 h3.c 引擎

antirez（Redis 作者）的项目，纯 C 宿主 + Metal 实现，MIT 许可，官方生态索引将其列为 Apple Silicon 的推荐路线。它不是通用推理框架：只服务 H3 一个模型、Apple Silicon 一个平台，省掉的是抽象层和兼容性的开销。

宿主侧 C 负责编排，GPU 侧两类路径：宽 BF16 矩阵乘与 SDPA 走 MPSGraph 缓存图，配直接 Metal 的正确性回退；其余 kernel 用运行时编译的 Metal 源码（继承自 antirez 的 Iris 项目），不依赖 Xcode 的离线 Metal 工具链——`make` 之后目录里的 `h3_shaders.metal` 必须随二进制保留，运行时按自身位置定位，就是为此。

芯片分层需要说清楚：README 的性能优化围绕 M3 Max 与 M5 Max 展开。Metal 4 TensorOps 的 int8 路径、GPU 侧采样器、safetensors 零拷贝映射都是 M5 专属；M1 走可移植路径——copied buffer、CPU 采样器、MPSGraph。功能等价，速度吃亏，4.4 节的性能预期按此打折。

引擎是带宽受限的，证据来自 h3.c 自己的 A/B：M5 Max 上固定 50 层、19 次过渡的 512×512 渲染，BF16 路径 36.30 秒、int8 路径 25.80 秒，同时峰值张量存储从 36.4 降到 25.9 GiB——权重流量减少直接兑换成时间减少，瓶颈在内存带宽而非算力。粗算可以验证量级：DiT BF16 约 61.7 GiB，每次前向至少完整读一遍，M1 Max 的 400 GB/s 下单次前向仅权重读取就有约 0.15 秒下限；默认 20 步、reuse 1 共 20 次前向，权重流量约 1.2 TiB，理想下限约 3 秒——未计激活内存、注意力计算与实际带宽效率，M1 的真实数字会明显高出这个下限。

### 2.3 内存策略

| 档位 | DiT 驻留（512×512） | 代价 |
|---|---|---|
| 全驻留（默认） | 约 36.5 GiB | 无 |
| `--ssd-streaming` | 约 2.0 GiB（864×480 约 2.1 GiB） | 前向慢 84%（512²）或 26%（864×480），输出逐位一致 |
| 全驻留 + `--layers 45/40` | 相应下降 | 少跑排序靠后的 transformer 块，画质换时间与内存 |

`--ssd-streaming` 的机制：仅每块的小型归一化权重常驻，两个 BF16 矩阵槽交替使用，后台线程按 checkpoint 偏移序预读下一块、与当前 GPU 命令缓冲并发执行，Darwin uncached read 避免文件系统缓存里再留一份拷贝。官方实测内置 SSD 读取吞吐 13–14.6 GiB/s。两个限制：不能与 `--use-int8-row-fc2` 同用（该选项 M5 专属，M1 本来也用不上）；交互式会话中用 `!ssd-streaming on` 开启。

两个数字容易被误读。2.0 GiB 只是 DiT 的被追踪张量存储，不含系统、媒体缓冲与输出分辨率的开销；提示词编码与两个 VAE 在独立阶段运行，各自峰值不与 DiT 阶段叠加。另外 `--show` 终端预览会让一个预览 VAE 常驻、额外增加约 10 GiB，低内存场景不要开。

分阶段加载是所有档位的底座：33B DiT、Qwen 编码器、解码器从不共存于统一内存，峰值由最大阶段决定。官方在 128 GB M5 Max 上给出的端到端 Ref2VA 实测：约 40.1 GB 物理占用、零交换。64 GB M1 Max 在 512×512 全驻留可行但余量有限，长片或更高分辨率建议直接流式。

交互式会话还有一项隐性收益：conditioning、准备好的 DiT 与视频解码器留在内存，同一提示词换 seed 复跑免重载、免重编码。

## 3. 引擎与权重

本章命令均在 zsh 下执行，沙盒根目录为 `~/H3-Sandbox`，引擎、权重、ComfyUI（5.2 节部署）全部收在里面，卸载等于删一个目录。

### 3.1 依赖与编译

目录规划：

```
~/H3-Sandbox/
├── h3.c/              # 引擎源码与二进制
├── weights/
│   └── MiniMax-H3/    # 权重，FL2VA/ 与 Ref2VA/ 的父目录
└── ComfyUI/           # 5.2 节部署
```

系统依赖两件：Xcode Command Line Tools（提供 cc 与 make）、ffmpeg 和 ffprobe。逐项确认：

```console
$ xcode-select -p
/Library/Developer/CommandLineTools
```

输出以上路径说明已安装。报 `command not found` 或弹出安装对话框，先执行 `xcode-select --install`，按引导装完再回来。

```console
$ ffmpeg -version | head -1
ffmpeg version ...
```

打印出版本行即可通过。未安装就 `brew install ffmpeg`（没有 Homebrew 先去 [brew.sh](https://brew.sh) 按首页命令装一个）。ffmpeg 是硬性要求，h3.c 的媒体输入与 MP4 输出都走它；确实无法放入 PATH 的场景，可用环境变量 `H3_FFMPEG`、`H3_FFPROBE` 指定绝对路径。

编译引擎：

```console
$ cd ~/H3-Sandbox
$ git clone https://github.com/antirez/h3.c.git
$ cd h3.c
$ make -j8
```

`make` 编译 C 源码并在当前目录生成 `h3` 可执行文件，过程中输出几十行编译日志，结尾没有 `error` 字样即成功。跑一下自检：

```console
$ ./h3 --help
```

能打印完整用法说明（`--profile`、`--ssd-streaming`、`--ref-image` 等 4、6 章用到的选项都在其中）即正常，这一步不依赖权重。可选 `make test` 跑主机侧确定性测试。

产物 `./h3` 与 `h3_shaders.metal` 必须留在原目录一起用：二进制运行时按自身位置查找 shader（2.2 节），把 `h3` 单独拷走会启动失败。

### 3.2 权重下载

**装 hf CLI。** Hugging Face 官方命令行工具，当前命令名是 `hf`，旧名 `huggingface-cli` 已废弃：

```console
$ uv tool install "huggingface_hub[cli]"
```

`uv tool` 把 `hf` 装进独立环境并链接到 `~/.local/bin`。装完若提示 PATH 警告，把 `export PATH="$HOME/.local/bin:$PATH"` 加进 `~/.zshrc`，重开终端再执行 `hf --help` 验证。没有 uv 就先 `brew install uv`。

已按 5.2 部署过 ComfyUI 的话，其 venv 内自带 `hf`——huggingface_hub 是 ComfyUI 的依赖——激活状态下（提示符带 `(.venv)`）可直接使用，代价是命令只在 venv 激活期间存在，断点续跑前需要先重新 `source` 激活。两种装法二选一。

**登录**（建议做，公开仓库匿名也能下，但匿名请求限流更严）：

```console
$ hf auth login
```

交互式流程，依次会问两件事：

1. 登录方式：方向键选 `Token`，回车
2. 粘贴 token：直接粘贴字符串，终端不回显（看不到任何字符是正常的），回车结束

token 去这里拿：浏览器登录 [huggingface.co](https://huggingface.co) → 右上角头像 → Settings → [Access Tokens](https://huggingface.co/settings/tokens) → New token，类型选 **Read**（下载只读权限就够），生成后立即复制——离开页面就不再显示明文。确认登录状态：

```console
$ hf auth whoami
```

会回显你的用户名。HF 网页打不开的话跳过登录，直接用下面的镜像方案下载。

**下载。** 先打开 [MiniMaxAI/MiniMax-H3](https://huggingface.co/MiniMaxAI/MiniMax-H3) 模型页过一遍许可条款（1.4 节）。只做文生视频，先下 FL2VA 一棵树，实测 82 个文件、144.1 GB（十进制；即 134 GiB），磁盘预留 160 GB 以上：

```console
$ hf download MiniMaxAI/MiniMax-H3 \
    --include "FL2VA/*" --include "model_index.json" \
    --local-dir ~/H3-Sandbox/weights/MiniMax-H3
```

参数用途：

- `--include` 圈定下载范围，只拉 FL2VA 子树和索引文件，避开根目录两套 61.7 GiB 的裸 transformer（1.2 节的 464 GiB 陷阱）。2.x 的 `hf` 每个 `--include` 只接受一个模式，多个模式重复写 flag；旧式 `--include "FL2VA/*" "model_index.json"` 写法里第二个模式会被解析成文件名，报 `File not found`（请求 URL 出现 `%2A` 即此症状）
- `--dry-run` 只列出将下载的文件与总大小，不产生下载流量，下载前核对范围用
- `--local-dir` 指定落盘位置；未完成的分片是 `<etag>.<随机后缀>.incomplete` 临时文件，存放在目标目录内 `.cache/huggingface/download/` 下，完成后消失，中断留下的旧分片不会被复用，可删

**传输后端与中断恢复。** venv 里的 huggingface_hub 2.4 自带 Xet 传输：hf_xet 已安装时默认走 Xet，`HF_HUB_DISABLE_XET=1` 强制回退普通 HTTP。实测速度差异明显——HTTP 单流随镜像波动，数 MB/s 至 200 kB/s；`HF_XET_HIGH_PERFORMANCE=1` 提高并发后实测 11.3 MB/s。

中断恢复是文件粒度，与后端无关：重跑同一命令，已完整落盘的文件直接跳过；中断时正在传输的文件从头再来。在传分片是进程唯一的临时文件（uuid 后缀，跨进程不复用），这是 2.4 的源码行为（`_download_to_tmp_and_move`，上游 PR #4228 的设计说明）。两点推论：弱网下 Xet 高性能模式完成同样剩余量耗时更短，中断重下的流量代价也更小；长时间中断后目标目录 `.cache/` 里会积累孤儿分片，删除即可回收空间。

**进度条读法。** 两行进度统计口径不同：`Fetching 82 files` 是文件数，`Downloading bytes` 与 `Reconstructing` 是字节数（多个在传文件的聚合，默认 8 个 worker 并行）。82 个文件里绝大多数是 KB 级的配置与源码，文件计数几分钟内冲到二十几之后长时间停滞——剩余容量集中在少数大文件上（video VAE 单文件 10.4 GB）。尾段进度以字节行和目录大小为准：

```console
$ du -sh ~/H3-Sandbox/weights/MiniMax-H3
```

之后要用参考生成（6 章）再补 Ref2VA，额外约 144 GB；根目录 `model_index.json` 已在盘上，无需重复 include：

```console
$ HF_ENDPOINT=https://hf-mirror.com hf download MiniMaxAI/MiniMax-H3 \
    --include "Ref2VA/*" \
    --local-dir ~/H3-Sandbox/weights/MiniMax-H3
```

完成后再跑一次 `--info`，清单里应多出 `Ref2VA DiT` 一行（约 61.7 GiB）。

**国内网络。** 直连 HF 不通时的两条替代路，两者都走国内网络：不需要代理，134 GiB 下载流量也不经过代理、不消耗代理配额。挂代理直连 HF 的代价是全部下载量计入代理流量，不在可选项里。

其一，hf-mirror 中转。给 `hf` 换一个端点，命令其余部分一字不改：

```console
$ HF_ENDPOINT=https://hf-mirror.com hf download MiniMaxAI/MiniMax-H3 \
    --include "FL2VA/*" --include "model_index.json" \
    --local-dir ~/H3-Sandbox/weights/MiniMax-H3
```

嫌每次敲前缀麻烦，把 `export HF_ENDPOINT=https://hf-mirror.com` 写进 `~/.zshrc`——但换回官方源或用 `hf auth` 时要记得注释掉。

其二，ModelScope 官方镜像，仓库 `MiniMax/MiniMax-H3`：

```console
$ uv tool install modelscope
$ modelscope download --model MiniMax/MiniMax-H3 \
    --include "FL2VA/*" \
    --local_dir ~/H3-Sandbox/weights/MiniMax-H3
```

注意与 hf 的参数名差异：`--model`、`--local_dir`。ModelScope 公开仓库无需登录；若个别命令提示要登录，执行 `modelscope login` 交互式配置，或干脆改走 hf-mirror。

另外 Wild H3C 文档提到部分 h3.c checkout 附带断点续传脚本 `download-h3.sh`（curl + hf-mirror，带重试）。clone 完 `ls download-h3.sh` 看一眼，有就用，没有不影响主流程。

**验证布局。** 下载完成后跑库存检查：

```console
$ cd ~/H3-Sandbox/h3.c
$ ./h3 --info -d ~/H3-Sandbox/weights/MiniMax-H3
```

`--info` 只读文件头、不加载权重，几秒内返回。M1 Max 64GB 上只下 FL2VA 时的实际输出：

```
h3-metal 0.1.0-dev
Device: Apple M1 Max (applegpu_g13s)
  physical memory       64.0 GiB
  recommended GPU set   51.8 GiB
  max Metal buffer      38.9 GiB
  Apple GPU family      7
  Metal 4               yes
  unified memory        yes
Native checkpoint inventory (header-only):
  Qwen3-VL encoder   14 files  1058 tensors   62.133 GiB
  FL2VA DiT          13 files   535 tensors   61.728 GiB
  Ref2VA DiT          0 files     0 tensors    0.000 GiB
  video VAE           1 files   560 tensors    9.700 GiB
  audio VAE           1 files  1087 tensors    0.564 GiB
```

四行组件与 1.1 节的构成表一致，Ref2VA 为 0 是未下载时的正常显示。这一步报错不要往下走，回下载命令核对缺了哪个子目录。正确布局：

```
MiniMax-H3/
├── model_index.json
└── FL2VA/
    ├── model_index.json
    ├── tokenizer/    text_encoder/    processor/
    ├── transformer/  video_vae/       audio_vae/
```

## 4. 命令行生成

### 4.1 首条视频与交互式会话

首条视频用最小配置验证链路：512×512、22 帧（约 0.9 秒）、4 步、不预览。第一次跑同时验证编译产物、权重布局、ffmpeg 管线三件事：

```console
$ cd ~/H3-Sandbox/h3.c
$ mkdir -p outputs
$ ./h3 --profile \
  -d ~/H3-Sandbox/weights/MiniMax-H3 \
  -p "A red fox walks through fresh snow in a pine forest. Medium tracking shot, natural winter light, realistic fur." \
  --width 512 --height 512 --frames 22 \
  --steps 4 --layers 50 --reuse 1 \
  -o outputs/first.mp4
```

行为预期：冷启动先等权重加载（37 GiB，无进度输出），随后终端滚动各阶段计时，结束生成 `outputs/first.mp4`——约 0.9 秒的狐狸雪地行走片段，带环境音。`--profile` 只增加计时报告不改生成路径，建议全程保留。M1 Max 64GB 实际输出（stderr 节选）：

```
tokenizer                    1/1
text encoder                50/50
refine text                  1/1
precompute AdaLN            50/50
load transformer core       50/50
denoise                      4/4
audio VAE                    7/7
video VAE load              36/36
FFmpeg                      22/22
h3: wrote outputs/first.mp4
```

stderr 里可能夹杂数行 `Failed to import MPS module`：Apple MPS 框架在 MPSGraph 初始化时输出的提示，h3 源码与二进制中均不含此字符串，三次出现分别对应 refine text、denoise、video VAE 三个阶段的 MPSGraph 初始化，生成不受影响。交互终端下该行被 `\r` 进度刷新覆盖，重定向到文件时才可见。

冒烟档的成片只用于验证链路：4 步去噪对参照档的画质损失显著（4.3 节，全片 SSIM 约 0.556），背景纹理与细节大量丢失是预期行为。评估模型真实画质用 20 步默认档。

README 示例里常见的 `--show`（终端内逐帧预览）这里刻意没加，两个原因：它要求 Kitty/Ghostty 或 iTerm2/WezTerm/Konsole 的图形协议，macOS 自带 Terminal.app 不支持；且预览 VAE 会额外常驻约 10 GiB（2.3 节）。调试没有 ffmpeg 的环境时，可以 `-o ''` 关掉 MP4 编码，配 `--frames-dir DIR` 直接落 PPM 帧文件。

**交互式会话**。去掉 `-p` 就进入 REPL，适合连续迭代：

```console
$ ./h3 -d ~/H3-Sandbox/weights/MiniMax-H3 --width 512 --height 512 --steps 6
```

直接输入提示词回车即生成，视频编号存入启动时打印的会话目录。会话把 conditioning、准备好的 DiT 和视频解码器留在内存——同一提示词换 seed 复跑免重载免重编码，这是它比反复起进程快的原因。常用命令：

| 命令 | 作用 |
|---|---|
| `!seed random` | 换随机种子 |
| `!seconds 2` | 改时长（自动对齐帧数） |
| `!status` | 看当前配置 |
| `!save output.mp4` | 保存最近生成 |
| `!show` | 回看已生成内容 |
| `!ssd-streaming on` | 会话内开流式 |
| `!help` | 全量命令 |

`!first` / `!last` / `!ref-image` 也是会话命令，用法在 6 章展开。

### 4.2 提示词

短句能跑，但官方系统按 Context-IR 风格的结构化描述训练，五要素齐了出片质量才稳：场景、动作、镜头、观感、声音。README 的示范：

```
Scene: a single red fox in a snow-covered pine forest at dawn.
Action: the fox walks steadily left to right and looks toward the camera once.
Camera: medium-height lateral tracking shot, 50 mm lens, stable framing.
Look: photorealistic fur, cold blue ambient light, warm sunrise rim light.
Audio: soft footsteps in snow, light wind through pine branches, no music.
```

身份与数量需要显式写出：写 "a single red fox" 才会得到单只主体，数量未声明时模型可能给出两只。声音是独立指令通道：H3 音画同出，Audio 行留空或写 no music 都是有效输入。seed 承担对比职能：`--seed` 默认 42，提示词、seed、画幅、帧数、步数全部一致时，结果差异才可归因于参数而非随机性。官方示例均为英文；模型卡附带 Base 与 Reference 两份提示词指南，Reference 规范（`<Picture N>`、对话标记）6 章用到时再展开。

### 4.3 参数矩阵

六个旋钮，前三个最常用：

| 旋钮 | 慢速参照 | 默认 | 激进 | 作用 |
|---|---|---|---|---|
| `--steps` | 50 | 20 | 4–7 | 去噪次数，写几就是几次 |
| `--reuse` | 1 | 1 | 3 | 整去噪器复用：20 步下实际前向 20 / 11 / 8 次（对应 1 / 2 / 3） |
| `--layers` | 50 | 50 | 40 | 生效的 transformer 块数，少跑的块权重不驻留 |
| `--core-reuse` | 1 | 4 | 6 | 每步刷新投影与输出头、少跑昂贵核心 |
| `--token-reduction` | 关 | 关 | 开 | 中段块成对合并水平 token，快但构图可能漂 |
| `--render-width/height` | 同输出 | 同输出 | 320² | 内部小画幅渲染后放大输出 |

README 认证的均衡档是 `--steps 20 --layers 45 --reuse 2`；Wild H3C 节点的 `balanced` preset 就是这一档（5.3 节）。步数 4–7 属低预算档，必须配 `--reuse 1`，保证每次请求的步数都真实执行；4 步结果对 29 步参照的全片 SSIM 约 0.556，去噪段在 M5 Max 上 3.5 秒对 26.4 秒。

硬性规则：

- `--reuse` 与 `--core-reuse` 互斥；砍层与两者皆可组合
- `--token-reduction` 不要与 `--layers 40 --reuse 3` 叠加——该组合实测产生色环、轮廓糊和肢体残影
- `--render-width/--render-height` 必须成对设置、与输出同长宽比、不大于输出尺寸；512 输出配 384 内部渲染可省 33% DiT 时间和 18% VAE 时间，320 是激进档
- 256×256 是原生快速预览档（自动半化空间 RoPE），但有效 token 网格只有 8×8，只配看构图，且该尺寸下关闭 token reduction；128² 不支持

### 4.4 性能预期

M1 Max 64GB 本机实测（512×512、22 帧、4 步、全驻留）：

| 阶段 | 耗时 | 峰值张量存储 |
|---|---|---|
| Qwen 文本编码 | 10.3 s | 2.7 GiB |
| DiT 加载 | 21.6 s | 36.2 GiB |
| 去噪（4 步） | 94.3 s | 36.2 GiB |
| audio VAE 解码 | 4.3 s | 0.3 GiB |
| video VAE 解码 | 18.0 s | 9.5 GiB |

全程约 2.5 分钟产出 0.9 秒成片，峰值 36.2 GiB——低于 macOS 默认 GPU 有线上限（64 GB 机型约 48 GiB），2.3 节的流式档位实测中无需启用。上表取自第二次运行（首次在去噪起点被中断，文件系统缓存已热），冷启动的加载段会再长一些。

与已发表数据点的对照：

| 芯片 | 配置 | 耗时 | 出处 |
|---|---|---|---|
| M1 Max 64GB | 512²、22 帧、4 步，去噪段 | 94.3 s | 本文实测 |
| M5 Max | 512²、22 帧、4 步，去噪段 | ≈3.5 s | h3.c README |
| M5 Max | 512²、22 帧、20 步 + layers 45 / reuse 2，去噪段 | 16.69 s | h3.c README |
| M5 Pro 64GB | 576×1024、124 帧（5.2 秒片）、BF16，全程 | 30 m 59 s | rznotes 实测 |
| M5 Pro 64GB | 同上，int8 | 23 m 59 s | rznotes 实测 |

同配置 4 步去噪相差约 27 倍，远超带宽差距（400 对 546 GB/s，约 1.4 倍）：M1 没有 M5 的 TensorOps int8 路径，MLP 与注意力落在 BF16 MPSGraph 上，GPU 核心规模也相差数代。按去噪段线性外推，20 步默认档约 8 分钟量级；124 帧、768 级画幅的正片为小时级。

测量方法沿用 README 的约定：同一配置连跑两三次取后者；机器发热时交替跑不同变体，避免把降频误判成参数差异。M1 Max 笔记本长时间满载会降频，插电、通风、垫高都会影响数字。

内存侧的对应手段是 `--ssd-streaming`：512² 全驻留实测峰值 36.2 GiB（与 2.3 节的 36.5 GiB 口径一致），64 GB 机器放得下；观察到内存交换时（生成中途系统卡顿、活动监视器出现 Swap 使用），流式模式以 26–84% 的前向时间换取稳定。

## 5. ComfyUI

### 5.1 是什么，和命令行的区别

ComfyUI 是开源的节点式工作流工具：一次生成被拆成「加载模型 → 提示词 → 参考图 → 生成 → 保存」等步骤，每一步是浏览器画布上的一个节点，节点之间的连线即数据流。它自身不包含 H3 的推理能力，Mac 上的 H3 节点来自 Wild H3C——对第 3 章编译的 h3 二进制的封装，引擎与命令行路线是同一个。

两条路线产出相同的视频，差异在操作形态：

| | 命令行（第 4 章） | ComfyUI（本章） |
|---|---|---|
| 上手 | 一条命令出片 | 部署加连线学习 |
| 参数 | 记 flag | 表单控件，随工作流文件保存 |
| 参考图、首尾帧 | 文件路径写进命令 | 拖入画布连线 |
| 复用 | shell 脚本 | 工作流文件，可分享 |
| 生态 | 无 | 社区节点（拼接、时间线、放大等） |
| 云端 | 无 | 官方 API 节点与本地节点同一画布（5.4 节） |
| 自动化接口 | shell 与 Agent 直接调用 | 经 ComfyUI API 间接驱动 |

素材接线频繁、需要与云端节点混排时画布省事；脚本化场景命令行是基础形态，第 7 章的自动化技能走的都是命令行或 API，不经过画布。两条路线在同一沙盒共存，互不排斥。

### 5.2 部署与 Wild H3C

ComfyUI 用独立 venv 部署，不污染系统 Python：

```console
$ cd ~/H3-Sandbox
$ git clone https://github.com/comfyanonymous/ComfyUI.git
$ cd ComfyUI
$ uv venv --python 3.12 .venv
$ source .venv/bin/activate
$ uv pip install -r requirements.txt
```

`uv venv` 创建虚拟环境，`source` 激活后提示符前面多一个 `(.venv)` 标记，之后的 Python 操作都在这个隔离环境里；以后每次使用先 `source ~/H3-Sandbox/ComfyUI/.venv/bin/activate` 再继续。Wild H3C 要求 ComfyUI 0.18+（V3 节点 schema，作者在 0.37.0 上验证过），直接 clone 主线必然满足。

装节点包。它只是 h3 CLI 的薄封装，不含引擎也不含权重，除 ComfyUI 自带依赖外零 Python 依赖：

```console
$ cd ~/H3-Sandbox/ComfyUI/custom_nodes
$ git clone https://github.com/skaiy/wild-h3c-comfyui.git
```

配置路径。仓库自带示例配置，复制一份再改：

```console
$ cd wild-h3c-comfyui
$ cp config.example.json config.json
```

编辑 `config.json`，填两个路径（支持 `~` 展开）：

```json
{
  "h3_binary": "~/H3-Sandbox/h3.c/h3",
  "model_dir": "~/H3-Sandbox/weights/MiniMax-H3"
}
```

`model_dir` 指向包含 `FL2VA/` 的目录本身，不要指进 `FL2VA/` 里面。每个节点还有高级输入 `binary_path` / `model_dir`，可按工作流临时覆盖，留空则读这里的全局配置。

启动：

```console
$ cd ~/H3-Sandbox/ComfyUI
$ source .venv/bin/activate
$ python main.py
```

终端滚动启动日志，看到 `To see the GUI go to: http://127.0.0.1:8188` 即就绪。这条命令会占住当前终端，Ctrl+C 停止服务。浏览器打开该地址，双击空白画布调出节点搜索，出现 Wild H3C 分类即安装成功；也可以在 ComfyUI Manager 里搜 Wild H3C 安装，效果相同。

首次执行的行为提前说明：队列一条任务后，前几十秒没有任何进度属正常——37 GiB 权重加载期间 h3 不打印进度行，加载完才进入去噪；h3 以 `\r<phase> done/total` 输出到 stderr，节点把它喂给 ComfyUI 进度条。节点对 h3 调用做了全局串行（一条进程已能打满 GPU），中断按钮会先 terminate 再等 5 秒 kill 子进程。

排错对照：

| 症状 | 处理 |
|---|---|
| `h3 binary not found` | 没编译或 `h3_binary` 路径错，回到 3.1 |
| `model directory not found` | `model_dir` 指错层级，回到 3.2 末尾的布局图 |
| ffmpeg / ffprobe 报错 | `brew install ffmpeg`，确认两者都在 PATH |
| 长时间无进度 | 权重加载中，等；不是卡死 |
| 旧工作流加载失败 | 节点 ID 发布时改名为 `WildH3*`，重新添加节点 |

卸载：删除 `~/H3-Sandbox` 与 `~/.cache/huggingface`（如无其他模型缓存）即可，系统层只留了 ffmpeg，`brew uninstall ffmpeg` 可选。

### 5.3 节点实战

ComfyUI 的价值在可视化编排：提示词、参考图、首尾帧、保存路径都成为节点连线，方便和后处理、拼接节点组合。Mac 本地跑 H3，这条路只有 Wild H3C 一座桥（上一节）。

最小工作流两个节点：**Wild H3 Text to Video** 的 `video` 输出直接接官方 **Save Video**，Queue 即开始生成。官方示例参数：`A drone shot gliding over a misty pine forest at dawn, cinematic.`，864×480、4 秒、20 步、preset 选 `balanced`。

节点输入逐项说明：

| 输入 | 说明 |
|---|---|
| `prompt` | 多行文本，原生 H3 提示词（4.2 节写法） |
| `width` / `height` | 32 的倍数，乘积 ≤ 768×1344，默认 864×480 |
| `seconds` | 0.25–15.0，24 fps 下自动对齐 5+17n 帧数 |
| `steps` | 去噪次数，默认 20 |
| `preset` | `precise` = layers 50 / reuse 1，`balanced` = 45 / 2（README 认证档），`fast` = 40 / 3 |
| `seed` | 64 位全范围，支持随机化 |
| `token_reduction`（高级） | 同 CLI 的 `--token-reduction` |
| `ssd_streaming`（高级） | 同 CLI 的 `--ssd-streaming`，64GB 机器建议保持开启 |
| `binary_path` / `model_dir`（高级） | 留空读 `config.json`，可按工作流覆盖 |
| `first_frame` / `last_frame` | 可选 IMAGE 输入，FL2VA 首尾帧锚定 |

**Wild H3 Reference to Video** 节点走 Ref2VA（需要 3.2 节补下第二棵树）：`ref_images` 接一批最多 9 张有序参考图，提示词里以 `<Picture 1>`、`<Picture 2>` 按序引用；`ref_audio` 接一段 2–15 秒音频，以 `<Audio 1>` 引用，且必须同时有至少一张参考图。引用输入与首尾帧锚定互斥，这是引擎层面的设计，不是节点限制。

迭代与定稿对应两组负载：小画幅（512×480 一类）+ `fast` preset + 低步数时单片在分钟级，承担构图试错；大画幅与高 preset 的耗时按 4.4 节的预期放大，属于定稿负载。节点对 h3 调用全局串行，队列中的任务逐条执行。

### 5.4 本地与云端

Wild H3C 文档给这套组合的定位很明确：本地免费迭代提示词、构图、参考；镜头锁定后，把最终提示词交给官方 MiniMax API 节点出终稿。官方 API 节点同时是使用未开放能力的唯一途径——闭源的 H3 Max 系列和 2K Regenerate 都只在云端。

接入比想象中简单：ComfyUI 内置这些 Partner Nodes，登录 Comfy 账号即可使用，按秒计费到 Comfy 账户的积分，全程不需要 MiniMax API key（对登录状态和网络环境有要求，见 [Partner Nodes 文档](https://docs.comfy.org/tutorials/partner-nodes/overview)）。模板库直接搜名字：基础 H3 三个模板（T2V / FLF2V / R2V）、H3 Max 四个、H3 Max Turbo 两个；跑全 Max 模板需 ComfyUI 0.34.3+，Turbo 需 0.34.4+，5.2 节 clone 主线自然满足。

三档云端模型：

| 模型 | 特点 | 官方费率 |
|---|---|---|
| H3（基础） | 直出最高 2K、5–15 秒 | 见平台价格页 |
| H3 Max | fal 后训，快于实时，提示词上限 5 万字符，480P / 768P | 15.09 credits/秒（480P）、24.14 credits/秒（768P） |
| H3 Max Turbo | Max 的约半价档，仅 T2V 与首尾帧 | 约 Max 费率一半 |

每个 Max 模板自带两个默认关闭的可选阶段，Switch 节点控制：**Prompt Enhance**（先用 Context IR 节点改写提示词再生成）和 **Upscale to 2K**（对首遍视频重渲染到 2K）。接线三条注意：Regenerate to 2K 只接受 768P 源，打算升 2K 就在 768P 生成；生成与提示词改写的时长输入必须一致，否则阶段漂移；额外的参考素材线要同时插到 Context IR 和 Regenerate 两个节点，它们不共享上下文。

费用量级（2026-11 时点，各平台随时调价，以官方页为准）：Comfy 侧 H3 Max 768P 每秒 24.14 credits，第三方折算约 $0.11/秒，5 秒片约 $0.55；fal 直连约 $0.08/秒（768P）；MiniMax 国内平台（minimax.cn）媒体报道价 768P 约 0.5 元/秒、2K 约 0.8 元/秒。15 秒正片在哪个平台都是十元上下的量级——对比本地 M1 的一小时，花钱买的是时间和 2K。

提示词规范随模型发布了两份官方指南，都在模型仓库里： [Base 模式提示词指南](https://huggingface.co/MiniMaxAI/MiniMax-H3/blob/main/docs/VIDEO_PROMPT_WRITING_GUIDE_base_en.md)与[参考模式指南](https://huggingface.co/MiniMaxAI/MiniMax-H3/blob/main/docs/VIDEO_PROMPT_WRITING_GUIDE_ref_en.md)，覆盖分镜计时、镜头运动、对话/音效/音乐的写法。参考模式的效果规律：引用被明确分派角色（身份、风格、动作、声音）时优于笼统描述；参考视频要求 23.976–60 fps。

## 6. 长片编排

本章与下一章是进阶层。H3 的原子输出是 15 秒以内的单段，完整影片由多段拼接而成，本章处理段与段之间的衔接和一致性，机制是引擎原生能力，命令行与 ComfyUI 两条路线通用。

### 6.1 引擎原生的衔接机制

H3 的原子输出是 4–15 秒，长片的全部问题归结为两件事：段间过渡不断裂、人物场景不跳变。前者靠首尾衔接，后者靠引用一致性。三种机制都是引擎原生能力，Mac 上开箱即用，不需要任何第三方节点。

**末帧接首帧。** 上一段的末帧经 ffmpeg 抽取后，作为下一段的 `--first-frame`：

```console
$ ffmpeg -sseof -0.1 -i outputs/clip1.mp4 -update 1 -frames:v 1 outputs/clip1-last.png
$ ./h3 --profile -d ~/H3-Sandbox/weights/MiniMax-H3 \
  -p "The fox keeps walking through the snow, then stops and looks up." \
  --width 864 --height 480 --frames 107 --steps 20 \
  --layers 45 --reuse 2 \
  --first-frame outputs/clip1-last.png \
  -o outputs/clip2.mp4
```

衔接质量取决于两个变量：下一段画幅与上段一致——首帧会被拉伸到目标画幅，比例不同会变形；段长落在 5+17n 帧档位——107 帧（4.458 秒）是常用的正片段长，帧数对齐后拼接处不抖。REPL 会话里锚定是持久的，`!first clip1-last.png` 设一次，之后每条提示词都带着这个锚生成，换段时 `!first clear`。

**视频引用续拍。** Ref2VA 路线（需要补下第二棵树），直接把上一段整段作为参考视频，模型延续其运动和声音：

```console
# 延续画面与运动，忽略原声
$ ./h3 -d ~/H3-Sandbox/weights/MiniMax-H3 -p "Continue the motion in this clip." \
  --width 864 --height 480 --frames 107 --steps 20 \
  --ref-silent-video outputs/clip1.mp4 -o outputs/clip2.mp4

# 保留原声续拍；或明确换配乐
$ ./h3 -d ... --ref-video outputs/clip1.mp4 ...
$ ./h3 -d ... --ref-video-audio outputs/clip1.mp4 replacement.wav ...
```

`--ref-video` 保留嵌入音轨，`--ref-video-audio` 接受一段替换音频——声音连续性靠这两个开关。

**角色一致性。** 同一角色跨镜头时，同一张参考图在每个镜头的生成中复用（`--ref-image`，提示词以 `<Picture 1>` 指代），一致性远高于提示词里的文字外貌描述。这是 Ref2VA 的核心用途，也是 1.1 节说它基础质量略低、第二棵树仍有下载价值的原因。

### 6.2 ComfyUI 成品工具与 Mac 边界

CUDA 阵营的长片工具已经成型，按机制列：

| 工具 | 机制 |
|---|---|
| ComfyUI-H3-Motion-Context（★491） | 把上一段的末帧**和音频**一起前馈给下一段，保持运动方向与速度；运行时校验 ComfyUI 版本，不匹配即拒绝启动 |
| joeygambino/MiniMax-H3-Multishot-Workflow | 多段 FL2VA/Ref2VA 串成连续序列，带音频交接，Apache-2.0 |
| ComfyUI-MiniMaxH3-Director（★182） | 时间线编辑器：轨道拖素材、标尺修剪、每镜头一段提示词、重拍、镜头串接 |
| comfyui-minimax-h3-audio-T8（★653） | 62 个节点含 Long Video 菜单，附带成套多镜头工作流 |
| H3 Auto Prompt Chain | 无值守多镜头链，给一份镜头清单自动逐段生成、可断点续跑；作者用它做过 2:47 的迷你纪录片 |

边界在于：这五件全部构建在 ComfyUI 官方 H3 节点（CUDA）之上，Motion-Context 甚至直接 patch 官方节点——Wild H3C 这座桥上目前没有验证可用的同类链。Mac 上的长片制作因此落在 6.1 的三种机制上：抽帧、锚定、引用、ffmpeg 拼接，由 shell 脚本串接成「拆分镜 → 逐段生成 → 串接」。这块空白是第 7 章 Agent 生态生长的位置。

### 6.3 配音与字幕

H3 自带的是环境音、音效和音乐——提示词 Audio 行控制的那部分。旁白和口播是另一层，两条路线。

本地免费：**Qwen3-TTS**，开源，MLX 实现跑在 Apple Silicon 上（M1–M4 均有报告），3–5 秒参考音频即可克隆声音，代表实现有 [qwen3-tts-apple-silicon](https://github.com/kapi2800/qwen3-tts-apple-silicon)（离线、含声音克隆与声音设计）和社区通用的 MLX-Audio 路线。产出 wav 后与视频轨在 ffmpeg 里合并。

云端高质量：MiniMax 自家 Speech 2.8（语音）和 Music 3.0（音乐）服务，与 H3 同厂，官方技能包把它们和视频生成打包成一套（见第 7 章）。

字幕与响度是纯 ffmpeg 的事：

```console
# 烧录 srt 字幕，音频直通
$ ffmpeg -i final.mp4 -vf "subtitles=narration.srt" -c:a copy final-sub.mp4

# 响度归一（多段拼接后各段响度不一致时）
$ ffmpeg -i mix.wav -af loudnorm mix-normalized.wav
```

音频后挂是更可控的成片结构：每段先出无声版定构图（`--ref-silent-video` 或提示词 no music），镜头全部定稿后统一配旁白与音乐，响度与风格的统一由后段处理决定，不依赖每段生成的随机音频。

## 7. Agent 化

第 4 到 6 章的每一步都是手动指令。本章的内容是把它们串成自动流程：现有技能包驱动什么、Mac 上的本地自动化长什么样、生态里哪些可信。

### 7.1 技能生态

官方生态索引单列了 Agent skills 一节，按完成度从高到低：

| 技能包 | 驱动对象 | 做到什么 |
|---|---|---|
| awesome-minimax-h3 | MiniMax 托管 API | 需求简报到成片文件全流程：提交、退避轮询、解析 file_id、ffmpeg 后处理；附带 Speech 2.8 与 Music 3.0 技能可组合。MIT |
| okaypic-video-skill | okaypic API | 多镜头成片：分镜表、角色与场景参考图、每镜头两次生成、字幕烧录（中英）、响度归一、可断点续跑。Claude Code / Codex 均可用。MIT |
| ComfyUI-Agent-Kit | 本地 ComfyUI | 从 Claude Code、Codex、Gemini CLI、Qwen Code 端到端驱动本地 ComfyUI：搭工作流、按硬件选模型、多镜头视频，581 个模板。Apache-2.0 |
| minimax-h3-opencode-skills | OpenCode | 导演、路由、分镜规划、提示词生成、审查的技能套件。MIT |
| Minimax-H3-Prompt-AgentSkill | 无 | 只产出格式化提示词，生成仍需自己跑 |

前两个走云端 API（费用结构见第 8 章），ComfyUI-Agent-Kit 走本地。awesome-minimax-h3 的技能文档里有一条现成的踩坑记录值得继承：官方文档的 `/v2/…/query` 轮询端点返回 404，实际可用的是 `GET /v1/query/video_generation`。

### 7.2 本地自动化

Mac 上的现实路径：h3.c 本来就是 CLI + REPL，天然可被脚本和 Agent 驱动。第 6 章的三种衔接机制全部是命令行操作，一个循环就能串起整条链：

```bash
for shot in shots/*.txt; do
  ./h3 -d ~/H3-Sandbox/weights/MiniMax-H3 \
    -p "$(cat "$shot")" \
    --width 864 --height 480 --frames 107 --steps 20 \
    --layers 45 --reuse 2 --ssd-streaming \
    --first-frame "frames/$(basename "${shot%.txt}").png" \
    -o "out/$(basename "${shot%.txt}").mp4"
  ffmpeg -sseof -0.1 -i "out/$(basename "${shot%.txt}").mp4" \
    -update 1 -frames:v 1 "frames/next.png"
done
```

这套流程写进 SKILL.md 后即可交给 Claude Code 一类 Agent 执行「拆分镜 → 逐段生成 → 抽帧衔接 → ffmpeg 拼接」；分镜规划与提示词书写是判断密集环节，也是 Agent 相对固定脚本的增量所在。ComfyUI-Agent-Kit 通过 ComfyUI API 驱动工作流，Wild H3C 节点同样是标准节点，这个组合在机制上可行，尚无公开案例。社区已有端到端先例：Reddit 上有用 Hermes Agent + ComfyUI + H3 把一条音轨做成音乐视频的完整记录。

### 7.3 成熟度与辨伪

这个生态新且热，教程质量参差，本文最初的需求同样来自一段真假混杂的对话。可用的校验信号有三类。

官方索引是第一道过滤。[awesome-minimax-h3-integration](https://github.com/MiniMax-AI/awesome-minimax-h3-integration) 由 MiniMax 官方维护，长片、加速、Agent 各分类齐全；不在这个索引里的工具首先存疑——早期教程里流传的 WorkBuddy-H3、TimelineDirector 两个名字，在官方索引、ComfyUI Registry 和社区讨论里均查无此项目，功能对得上的实际存在物是 7.1 表里的技能包和 6.2 表里的节点。

驱动方式是第二个信号。走标准 SKILL.md / MCP / ComfyUI API 的技能可审计、可在本地复跑；走私有 API 代理的（okaypic 属此类）成本结构不透明。

「一句话出成片」的承诺对应的是第 6 章那套拆段与衔接机制——机制固定，工具之间的差别在封装完整度与踩坑多少。

## 8. 费用

本章集中算清前文散落的费用问题：本地链路的真实成本、云端价目、三类典型场景的对比。

### 8.1 本地要不要钱

不要。逐项清点：h3.c 引擎 MIT，Wild H3C 节点 MIT，ComfyUI 开源免费，权重对适用区域内的个人使用免费。5.1 节的全流程——文生视频、首尾帧锚定、参考生成、长片拼接——零边际成本。

会产生费用的只有两类，都不在本地链路上：一是 5.4 的云端增强（H3 Max 生成、Context IR 提示词改写、2K Regenerate、云端 TTS 与音乐），全部可选；二是本地生成内容用于商业用途时需向 MiniMax 购买商业许可（Comfy 代售，见 1.4 节），这是授权费不是使用费。

本地的真实成本是时间和电：M1 Max 满载整机功耗百瓦量级，按民用电价粗算一小时电费几分钱，与云端按秒计费相比可以忽略。真正的成本是 4.4 节那个小时级的等待。

### 8.2 云端价目

| 渠道 | 计费方式 | 2026-11 参考价 |
|---|---|---|
| Comfy Partner Nodes · H3 Max | credits/秒 | 480P 15.09、768P 24.14（官方价） |
| Comfy Partner Nodes · H3 Max Turbo | credits/秒 | 约 Max 的一半（官方口径） |
| fal 直连 | 按秒按分辨率 | 768P 5 秒片 $0.40 |
| MiniMax 国内平台 | 按秒 | 768P 约 0.5 元/秒、2K 约 0.8 元/秒（媒体报道价） |

价格随时调整，以各平台价格页为准；Context IR 与 2K Regenerate 单独计费，费率见 [Partner Nodes 价格表](https://docs.comfy.org/tutorials/partner-nodes/pricing)。

### 8.3 三个场景算账

**场景 A：单条 5 秒 768P 正片。** 本地：一小时上下的生成时间，电费几分钱。云端：Comfy Max 约 120 credits（三方折算约 $0.55）；fal $0.40；国内平台 2.5 元。

**场景 B：60 秒长片**（六段 10 秒，243 帧档）。本地：六段小时级生成，基本是一个通宵，电费不到一元。云端：国内平台 768P 60 × 0.5 = 30 元；Comfy Max 约 $6.6；若再开 2K Regenerate，费用另加。

**场景 C：迭代期，一天 50 次构图试错**（每次 5 秒小样）。本地：0 元；云端：50 × 2.5 = 125 元/天。迭代次数是云端费用的乘数，本地迭代成本恒为零——「本地迭代、云端终稿」这一官方推荐模式的算术依据在此，也是慢机器上的工作流依然成立的根基。

三个场景的数字指向同一结构：本地成本以时间计，云端成本以次数计。M1 的特殊性在时间长——一小时的等待挡住发布节奏时，30 元买回的是这段时间外加 2K；自用场景下，通宵挂机的边际成本为零。

## 参考

取证时间 2026-11，各项目均处活跃开发期，参数与价格以当日上游为准。

**官方**

- [antirez/h3.c](https://github.com/antirez/h3.c) — 引擎本体。编译方式、CLI 全部参数、内存三档、帧数与画幅规则、参考生成机制、M5 系列性能数据均出自其 README
- [skaiy/wild-h3c-comfyui](https://github.com/skaiy/wild-h3c-comfyui) — ComfyUI 桥接节点。节点参数表、config.json、排错表出自其 README
- [MiniMaxAI/MiniMax-H3](https://huggingface.co/MiniMaxAI/MiniMax-H3) — 权重仓库。文件构成、两份提示词指南（`docs/VIDEO_PROMPT_WRITING_GUIDE_base_en.md`、`..._ref_en.md`）、Community License
- [MiniMax-AI/awesome-minimax-h3-integration](https://github.com/MiniMax-AI/awesome-minimax-h3-integration) — 官方生态索引。长片工具、Agent 技能、Ref Patch、量化与加速生态的选型依据
- [ComfyUI Partner Nodes · MiniMax H3](https://docs.comfy.org/tutorials/partner-nodes/minimax/minimax-h3) — 云端节点文档。H3 Max 三档模型、官方费率、模板清单与接线约束
- [Partner Nodes 价格表](https://docs.comfy.org/tutorials/partner-nodes/pricing) — 各云端节点计费明细
- [comfyanonymous/ComfyUI](https://github.com/comfyanonymous/ComfyUI)

**社区实测与数据**

- [MiniMax H3 on Mac: Files, Software and Timings](https://modelfit.io/blog/minimax-h3-on-mac)（modelfit.io，2026-10-24）— Mac 各路线实测数据汇总：M5 Pro 64GB 的 h3.c 31 分钟数据、MLX 4-bit 峰值内存、Argus 与 Draw Things 基准、许可条款引文
- rznotes 的 M5 Pro 64GB 实测（h3.c 与 ComfyUI 两条路线，2026-08-13，经上文汇总）
- [fal.ai · MiniMax H3 Max](https://fal.ai/minimax-h3-max) — fal 直连价格样本

**Agent 与音频**

- [kapi2800/qwen3-tts-apple-silicon](https://github.com/kapi2800/qwen3-tts-apple-silicon) — Qwen3-TTS 的 Apple Silicon 本地实现
- [Qwen3-TTS with MLX-Audio on macOS](https://mybyways.com/blog/qwen3-tts-with-mlx-audio-on-macos) — MLX-Audio 路线配置记录
- [Hermes Agent + ComfyUI + H3 音乐视频](https://www.reddit.com/r/StableDiffusion/comments/1vlllh2/minimax_h3_comfyui_hermes_agent_music_video/) — Agent 端到端成片实例
- [H3 Auto Prompt Chain 发布帖](https://www.reddit.com/r/comfyui/comments/1vpw6hp/i_made_a_simpler_way_to_run_long_minimax_h3/) 与[两分四十七秒纪录片后续](https://www.reddit.com/r/comfyui/comments/1vknr0v/followup_from_a_5second_clip_to_a_247/) — 无值守多镜头链的真实产出

MiniMax 国内平台价格（768P 0.5 元/秒、2K 0.8 元/秒）为 2026-11 媒体报道口径，未经官方页面核实。
