# 03_数据 —— 数据集清单与获取方式

本目录为实验所需的**全部原始数据集与特征缓存**（共约 5.9 GB），通过 GitHub Release 分卷分发。
完整获取方式见 `08_交付件/05_标注数据集交付说明.md` §1.1，
仓库地址：**https://github.com/zixilee666-svg/encrypted-traffic-dpi**。

## 目录结构

```
03_数据/
├── cesnet/       13 个按日切分的 parquet（D20220213 … D20221228）+ cache/（特征矩阵缓存）
├── ustc/         USTC-TFC2016 原始 pcap（Benign/ 12 类 + Malware/ 8 类）
├── quic/         CESNET-QUICEXT25 的 9 个按日 parquet（D20240603 … D20241003）
├── mirage/ccma/  MIRAGE-COVID-CCMA-2022（SEQ pickle）
├── darknet2025/  SafeSurf Dataset Layer 1/2&3
├── lens/         Lens / NetBench 派生跨平台数据集（13 个子任务）
└── E1_WAN_CohortB_npz/  自采私有协议集（75 npz + 75 manifest）
```

| 数据集 | 用途 | 体积 | 官方/公开来源 |
|---|---|---|---|
| CESNET-TLS-Year22 | TLS 分类主矩阵 + 真实年度漂移（171/180 类） | 1.22 GB（含缓存 328 MB） | `https://hf-mirror.com/datasets/Lystea/CESNET-TLS-YEAR22-PARQUET` |
| CESNET-QUICEXT25 | QUIC 跨协议通用性（53 类） | 441 MB | `https://hf-mirror.com/datasets/Lystea/CESNET-QUICEXT25-PARQUET` |
| USTC-TFC2016 | 真实 PCAP 端到端 + 规则编译吞吐（14 类可用） | 4.20 GB / 24 个 pcap | `https://github.com/davidyslu/USTC-TFC2016` |
| MIRAGE-COVID-CCMA-2022 | 精细化行为（9 IM × 语音/视频/聊天） | 106 MB（SEQ pickle） | `http://traffic.comics.unina.it/mirage/` |
| SafeSurf Darknet-2025 | 真实匿名协议流（Tor/I2P/Freenet/Zeronet/VPN + 8 类行为） | 181 MB | Mendeley Data |
| Lens / NetBench 派生集 | 跨平台 Android/iOS 分类（13 个子任务） | 117 MB | `Charles59/lens-network-traffic`（HF） |
| E1_WAN_CohortB_npz（自采） | 私有代理协议公网层证据 | 2.9 MB（75 npz + 75 manifest） | 自建采集管线 |

> **数据来源声明**：本项目全部训练与测试数据均来自上述公开发布的数据集与自采流量，
> 不依赖赛方私有数据。公开数据集遵守各自许可，未做任何标签篡改或样本拼接。

## 为什么内置数据集文件

1. 官方 CESNET 服务器长期不可达，实测用的是 Hugging Face parquet 镜像；换机器重跑要重走一遍 30+ 分钟下载与 20 分钟缓存。
2. USTC 镜像共 4.2 GB，重下成本高。
3. `cache/*.npz` 是 `p02_cache.py` 的产物（blake2b 稳定哈希标签），内置后可**跳过 S2 直接复查结果**。
4. `mirage/` 只保留 p22 实际使用的 SEQ pickle（106 MB）；CCMA 原始 zip（3.06 GB）、载荷 pickle（998 MB）未内置，
   重跑 `p08_fetch_new.py mirage` 即可获得完整件。

## 换机器/换盘符怎么办

所有脚本的数据根目录已参数化为环境变量 `ETC_DATA`（默认值仍是 `D:/etc_data`）：

```bash
# Windows PowerShell
$env:ETC_DATA = "D:\你的路径\加密流量特征挖掘与DPI识别_完整包\03_数据"
python 06_复现/run_all.py

# bash / Linux / macOS
export ETC_DATA=./加密流量特征挖掘与DPI识别_完整包/03_数据
python 06_复现/run_all.py
```

`06_复现/run_all.py` 会自动探测：若 `D:/etc_data` 不存在而本包 `03_数据` 有内容，自动指向本包目录。

## 数据完整性自检

```bash
python -c "import os;print({d:sorted(os.listdir('03_数据/'+d))[:3], len(os.listdir('03_数据/'+d))) for d in os.listdir('03_数据')})"
# 期望：cesnet 14 项（13 parquet + cache）、quic 9 项、ustc 2 项（Benign/Malware）
```

## E2 跨 OS 验证数据说明

E2 使用 `Charles59/lens-network-traffic` 的 Cross Platform 派生公开真实数据，按 Android / iOS 同名应用进行人工映射与支持度筛选。正式主层级固定为 **support≥10、K=8**：ESPN、QQ、QZone、Renren、Talking Tom、Weibo、Youku、Zhihu。

实验协议：

- target TEST 全程冻结，只在最终模型确定后评估一次；
- 模型/超参数仅在 target validation 内进行分层交叉验证；
- packet-motif 表示仅使用方向、协议、粗粒度包长、TCP flags 与包序；**不使用 IP、端口、host/domain 或 payload**；
- E2-v3 使用 source + 全部 target validation 标签，因此结果属于 **target-calibrated cross-OS adaptation**，不是 zero-shot；严格 5-shot 证据保留在 E2-v2。

主结果：

| 方向 | Test Accuracy | Macro-F1 | Target-only calibration ceiling |
|---|---:|---:|---:|
| Android→iOS | **0.5771** | **0.5059** | 0.5871 / 0.5404 |
| iOS→Android | **0.7040** | **0.6158** | 0.7090 / 0.6260 |

判定：`PASS_WITH_TARGET_CALIBRATION`。证据文件位于 `04_结果/E2_cross_os_v3_max/`。
