# BiliDigest

B 站视频字幕提取与 AI 摘要工具。基于 bili2text 转写引擎，提取视频字幕并生成结构化摘要。当前摘要模板固定为财经市场方向，后续可扩展。

## 功能

- **单视频解析**：输入 B 站 BV 号，自动下载音频、转写字幕并保存
- **AI 摘要**：基于大模型生成视频观点摘要，含事实核实机制（校准股票名/数字/观点归属）
- **错峰调度**：DeepSeek 峰谷定价感知，高峰自动入队、低谷自动消费队列
- **定期跟踪**：管理 UP 主列表，批量解析最新视频字幕
- **配置管理**：自定义 bili2text 路径、B 站 Cookie、调试日志路径
- **大模型集成**：支持 DeepSeek 和火山方舟（豆包），可在配置页切换
- **系统托盘**：点击关闭或最小化按钮时自动隐藏到系统托盘，托盘菜单提供「显示 / 退出」

## 快速开始

需系统预装 Python 3.11：

```powershell
git clone git@github.com:cementbarrier/stock-tool.git
cd stock-tool
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## 使用

```powershell
# 直接运行
.venv\Scripts\python.exe gui\build\gui.py

# 打包为 EXE（使用 gui.spec）
.venv\Scripts\python.exe -m PyInstaller gui.spec
```

打包输出至 `dist\gui.exe`。

首次使用请在**配置页**中设置 bili2text 路径和 B 站 Cookie。

### bili2text 安装（系统需单独安装）

bili2text 是核心转写引擎：

```powershell
git clone https://github.com/lanbinshijie/bili2text.git D:\bili2text
cd D:\bili2text
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Cookie 文件默认位于 `D:\bili2text\.b2t\cookies.txt`。

### 大模型配置

在配置页「大模型」分区设置：

| 字段 | 说明 |
|------|------|
| 提供商 | 可选 DeepSeek 或火山方舟/豆包 |
| API Key | 对应平台的 API 密钥 |
| 模型名 | deepseek-v4-pro / 豆包 lite/mini |

配置完成后点击「保存配置」。

## 项目结构

```
stock-tool/
├── gui/
│   ├── build/
│   │   ├── gui.py                  # 主界面（Tkinter，含托盘图标/日期选择器）
│   │   ├── pages/
│   │   │   ├── page_single.py      # 单视频解析页
│   │   │   ├── page_batch.py       # 定期跟踪页
│   │   │   └── page_config.py      # 配置页
│   │   └── assets/
│   │       └── app_icon.ico        # EXE 图标
│   └── gui.spec                    # PyInstaller 打包配置
├── backend/
│   ├── config_manager.py           # 配置管理（settings.json 读写/加密）
│   ├── llm_client.py               # 大模型统一接口（DeepSeek/火山方舟）
│   ├── single_parser.py            # 单视频解析
│   ├── batch_parser.py             # 批量解析
│   ├── up_manager.py               # UP主列表管理
│   ├── single_summary_client.py    # 单视频AI摘要（含事实核实）
│   ├── task_queue_manager.py       # 错峰任务队列
│   ├── time_price_judge.py         # 峰谷时段判断
│   ├── valley_scheduler.py         # 低谷调度器
│   └── parsed_records.py           # 已解析记录管理
├── scripts/                        # 流水线脚本（step1~5）
├── data/                           # 运行时数据（parsed_records.json）
├── config/                         # 配置文件
├── requirements.txt
└── .gitignore
```

## 已修复问题概览

共修复 42 项问题，涵盖：托盘与窗口（4）、配置持久化（5）、GUI 显示（3）、构建部署（4）、PyInstaller 冻结环境（5）、批量解析（8）、日期选择器（6）、配置页模型（3）、外观（4）。详见 [SESSION_CONTEXT.md](SESSION_CONTEXT.md)。

## 许可证

MIT
