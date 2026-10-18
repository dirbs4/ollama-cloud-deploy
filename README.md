# Ollama云服务器部署：配置怎么选、Docker 怎么装、远程访问怎么配才安全

把 Ollama 搬上云服务器，部署本身是最简单的环节：官方提供 Linux 一键脚本和 Docker 镜像，一台普通 VPS 十来分钟就能把 API 跑起来。真正决定体验和成本的是另外两件事——配置怎么选才不白花钱，以及怎么让 11434 端口只对你开放、不暴露给整个互联网。

这篇按“选配置 → 开机器 → 部署 → 远程访问 → 排错”的完整流程走一遍。服务器以 Vultr 为例，原因是它按小时计费、随时销毁重开，试错的账很好算；流程本身在任何一家的 Linux VPS 上都通用。

> **选购速览**：大多数人拿云服务器跑 Ollama，选 4 核 8GB 内存的 Cloud Compute 档（$40/月）就能跑 7-8B 量化模型；只跑 1-3B 小模型，$5-10/月足够；想上 32B 以上，直接考虑 32GB 内存或 GPU 实例。全系按小时计费，开错了删掉换一台就行。👉 [查看 Vultr 全部套餐和当前价格](https://www.vultr.com/pricing/?ref=7268283)

## 内存是第一指标：配置怎么选才不浪费钱

Ollama 本身很轻，真正吃资源的是模型。一条实用的估算基准：8-bit 量化下，每 10 亿参数大约需要 1GB 内存，4-bit 量化减半。也就是说 7-8B 的模型，8-16GB 内存的机器不带 GPU 也能跑；参数过了 30B，没有 GPU 就只能靠耐心了。

很多教程只教你看模型文件大小，这里补一个容易被忽略的变量：Ollama 默认上下文窗口是 4096 tokens，上下文开得越大，KV 缓存占的内存越多。模型刚好装下、一问长问题就崩的情况，多半栽在这里。调上下文可以用 `OLLAMA_CONTEXT_LENGTH` 环境变量，或者在 API 调用时指定 `num_ctx` 参数。

给系统和你想加的上下文长度预留 20-30% 余量，就不会反复重开机。

## Vultr 套餐怎么挑：先看常规档位，再决定要不要 GPU

结论先行：7-8B 量化模型选 $40/月的 4 核 8GB；14B 选 $80/月的 16GB；32B 选 $160/月的 32GB；再往上，CPU 路线的性价比撑不住，该看 GPU 实例。

下面是目前官网常规 Cloud Compute 系列的全部档位（月付价格即封顶价，实际按小时累计）：

| 套餐 | 月付 | vCPU | 内存 | 存储 / 流量 | 开通 |
| --- | --- | --- | --- | --- | --- |
| Cloud Compute 512MB | $2.50 | 1 | 512MB | 10GB SSD / 0.5TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 1GB | $5.00 | 1 | 1GB | 25GB SSD / 1TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 2GB | $10.00 | 2 | 2GB | 55GB SSD / 2TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 4GB | $20.00 | 2 | 4GB | 80GB SSD / 3TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 8GB | $40.00 | 4 | 8GB | 160GB SSD / 4TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 16GB | $80.00 | 6 | 16GB | 320GB SSD / 5TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 32GB | $160.00 | 8 | 32GB | 640GB SSD / 6TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 64GB | $320.00 | 16 | 64GB | 1.28TB SSD / 7TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 96GB | $640.00 | 24 | 96GB | 2.56TB SSD / 8TB | [ 开通](https://www.vultr.com/pricing/?ref=7268283) |

计费按小时累计、以月价为上限。拿 $40 档折算，约 $0.06 一小时，开一晚上试模型花费不到 1 美元。新用户注册一般还有一笔 30 天内有效的试用赠金（长期是 $250 额度，具体金额以注册页显示为准）。机房覆盖 30 多个地区，选离你或你的用户近的节点即可。

除了常规系列还有几条产品线：High Performance 用 EPYC 处理器配 NVMe，同规格比常规档贵几美元、起步 $6/月，内存带宽更好，CPU 推理速度一般也更快；High Frequency 主频更高，起步价同样是 $6/月；要真 GPU 的话有 Cloud GPU 系列，NVIDIA 卡按小时计费，跑 30B 以上模型再考虑它。

顺手交代一下其他路线，免得多花冤枉钱：Oracle Cloud 永久免费层里有 ARM 实例额度（合计 4 核 24GB），试水小模型不花钱，但 CPU 推理速度摆在那里；RunPod、Vast.ai 这类平台专做 GPU 短租，适合临时跑大模型；LM Studio 更适合有屏幕的桌面机，headless 服务器是它的短板；想抠到每一层控制，可以直接裸跑 llama.cpp，代价是参数全要自己管。

16GB 内存这个档位是 14B 量化模型的门槛，也是很多人实际会停下来的位置：👉 [选一台 16GB 内存的云服务器](https://www.vultr.com/pricing/?ref=7268283)

## 两种部署方式：一键脚本和 Docker

选一条就够：想最快跑起来用一键脚本，想把服务和数据管得干净用 Docker。机器还没开的话先开一台：👉 [注册 Vultr，按小时计费先跑起来](https://bit.ly/VulTR)

### 路线一：官方脚本

bash
curl -fsSL https://ollama.com/install.sh | sh


装完会自动注册成 systemd 服务并启动。拉一个模型进对话试试：

bash
ollama run qwen3:8b


第一次运行会自动下载权重，之后模型就在服务器上常驻了。

### 路线二：Docker

bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
docker exec -it ollama ollama run llama3.2


数据卷 `ollama` 存模型和配置，容器删了模型不丢。有 NVIDIA GPU 加 `--gpus=all`；AMD 卡换用 rocm 标签的镜像。验证 API 是否通：

bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4",
  "messages": [{"role": "user", "content": "你好"}],
  "stream": false
}'


## 远程访问与安全：别把 11434 端口裸奔到公网

默认配置下 Ollama 只监听 127.0.0.1，你从本地是连不上的；要连就得动监听地址和防火墙。顺序建议反过来：先选好访问方式，再决定要不要改监听。

**方案一（推荐）：SSH 隧道，零改动。** 服务端什么都不用配，本地执行：

bash
ssh -L 11434:localhost:11434 root@你的服务器IP


然后本地应用直接连 localhost:11434，等于连到了服务器上的 Ollama，公网完全摸不到 API。

**方案二：改监听 + 防火墙白名单。** 需要多设备直连时，用 systemd 编辑服务：

bash
systemctl edit ollama.service


加入：


[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"


保存后执行 `systemctl daemon-reload && systemctl restart ollama`。同时务必在防火墙上把 11434 限制为只对你的 IP 放行，Vultr 控制台自带 Firewall 功能，可以先建规则组再绑定到实例。

**方案三：反向代理。** 要给别人用或挂域名，在前面加一层 Nginx 之类的反代，套上 HTTPS 并加认证，别让 API 以明文形式直接暴露。

Docker 用户注意一个坑：`-p 11434:11434` 会把端口绑定到服务器的 0.0.0.0，防火墙没拦的情况下等于把 API 直接挂上了公网。只想走本地隧道访问的话，改成 `-p 127.0.0.1:11434:11434` 更稳。

模型拉取如果受网络环境限制需要走代理，只设 `HTTPS_PROXY`，别设 `HTTP_PROXY`——官方文档明确提示 Ollama 拉模型只走 HTTPS，设置了 HTTP_PROXY 反而可能把连接弄断。

服务器场景常被问到的隐私问题也一并交代：Ollama 本地运行时，提问和回答不会回传；如果连它的云功能和联网搜索都不想要，在 `~/.ollama/server.json` 里把 `disable_ollama_cloud` 设为 true，或者设环境变量 `OLLAMA_NO_CLOUD=1`，重启后就是纯本地模式。

## 部署完先做两个检查

第一个，用 `ollama ps` 看模型加载状态。Processor 列显示 100% GPU 是整卡加载，100% CPU 是纯内存推理，48%/52% CPU/GPU 是混合加载。无 GPU 的云服务器上显示 100% CPU 属于正常现象，不代表装错了。

第二个，确认模型选型。Qwen、DeepSeek、Gemma、gpt-oss、GLM 这些主流开源家族在 Ollama 的模型库里都有，`ollama run` 一行命令拉取，模型名字就是你要操作的全部——不用管量化版本、模板和启动参数，Ollama 替你处理了。参数规模从不到 1B 到几百 B 都有，决定上限的始终是机器的内存。

## 常见问题

**没有 GPU 的云服务器跑 Ollama 有意义吗？**
有，看用途。7-8B 模型在 8-16GB 内存的机器上可用，拿来做 API 后端、接自动化脚本和网页界面完全够；实时生成长文的速度取决于 CPU 和内存带宽，高频聊天式使用会觉得慢。

**部署完怎么接网页界面？**
社区最常用的搭配是 Open WebUI，一个自托管的聊天前端，连上 11434 端口就能用。也可以直接用 API——Ollama 的接口风格接近 OpenAI，现有工具迁移成本很低，官方还提供 Python 和 JavaScript 库。

**上下文窗口怎么改？**
默认 4096 tokens。用 `OLLAMA_CONTEXT_LENGTH=8192 ollama serve` 可以全局调整，调用 API 时用 `num_ctx` 参数单独指定。内存紧张的机器别开太大，KV 缓存会吃掉余量。

**不想续费了怎么办？**
按小时计费的机器，销毁实例即停止计费。模型文件存在实例盘和数据卷里，会跟着实例一起消失，想保留先做快照，之后开新机器可以恢复着用。

## 按用途定配置

- 只想试水、跑 1-3B 小模型 → $5-10/月档位足够，甚至先按小时开一台玩几天再决定
- 7-8B 日常问答、当 API 后端 → $40/月的 4 核 8GB 是甜点位
- 14B 到 32B 量化模型 → $80-160/月，建议直接上 High Performance 系列
- 70B 级别或要求响应速度 → 别跟 CPU 较劲，看 Cloud GPU

下单前确认两件事就够了：目标模型的内存需求你算过余量；11434 的暴露方式按上面配好。然后就可以 👉 [去 Vultr 开一台服务器，把 Ollama 跑起来](https://bit.ly/VulTR)
