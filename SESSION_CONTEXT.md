# BiliDigest 项目上下文（2026-07-25）

BiliDigest——B 站视频字幕提取与 AI 摘要工具。Tkinter GUI，基于 bili2text 转写引擎，当前摘要模板固定为财经市场方向。

将以下内容粘贴到新对话开头，AI 即可接续工作。

---

## 项目基本信息

- **项目路径**：`E:\stock-tool`
- **GitHub**：`git@github.com:cementbarrier/stock-tool.git`（SSH）
- **入口文件**：`gui/build/gui.py`（Tkinter GUI）、`scripts/run_pipeline.py`（定期跟踪流水线）
- **桌面快捷方式**：`stock-tool.lnk` → `E:\stock-tool\dist\gui.exe`
- **Python 运行环境**：项目 venv `E:\stock-tool\.venv`（由 Marvis 自带 Python 3.11.8 创建）

## 部署流程（极其重要）

每次修改源码后走完整流水线：

```
1. 杀进程: Stop-Process -Name gui -Force -ErrorAction SilentlyContinue
2. 清理: delete 工具移 build/ 和 dist/ 到回收站
3. 构建: E:\stock-tool\venv\Scripts\python.exe -m PyInstaller E:\stock-tool\gui.spec（用 python_executor，timeout=900）
4. 验证: EXE 时间戳必须晚于所有源文件
5. git add -A && git commit -m "..." && git push（commit message 避免中文标点）
6. 重建桌面快捷方式: stock-tool.lnk → E:\stock-tool\dist\gui.exe
```

**关键约束**：
- 构建用 venv Python，不用系统 Python
- gui.spec 含 `upx=False`（关闭 UPX 压缩以缩短构建时间）
- 开发循环不复用 `--clean`，保留 build 缓存加速迭代
- 按钮点击后不可用改为灰色禁用 `state="disabled", fg="#AAAAAA"`，不要 `place_forget()`
- 保留用户认可的既有视觉风格，不要擅自套用统一新配色

## 全部已修复问题（共 42 项）

### 托盘与窗口
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 1 | 托盘图标左键点击后消失 | `_hide_to_tray` 图标为空时不重建 | 图标为空时调用 `_init_tray_icon` 重建 |
| 2 | 关闭窗口后无托盘图标 | 同上 | 同上 |
| 3 | 托盘图标不应随窗口显示/隐藏而销毁 | `_restore_window` 调用了 `_tray_icon.stop()` | 移除 stop 调用，图标永不销毁 |
| 4 | 系统 12 个 gui.exe 进程堆积 | 托盘图标消失后用户误以为已关，反复双击 | 启动时文件锁 `stock_tool_instance.lock` 单实例检测 |

### 配置持久化
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 5 | 提供商/API Key/模型名更新软件后丢失 | `_config_save_all` 中 `if selected:`/`if api_key:`/`if model:` 的 falsy 守卫跳过空值写入 | 移除守卫，改为全量写入 |
| 6 | 切换提供商后模型被清空 | `_config_provider_changed` 清空了模型 | 改为设默认模型 |
| 7 | 邮件三字段保存后重启仍为空 | 逐字段 `set_setting` 每次独立读写 JSON，与 Checkbutton 自动保存回调交错 | 改为一次 `load_settings` → 改全部 → 一次 `save_settings` 原子写入 |
| 8 | `load_settings` 把空字符串 `""` 判为缺失回退默认值 | `data.get(key) if data.get(key) else DEFAULTS[key]` 把空字符串判为 falsy | 改为 `val if val is not None else DEFAULTS[key]` |
| 9 | `settings.json` 带 UTF-8 BOM 头导致 JSON 加载失败 | `json.load` 用 `utf-8` 编码碰到 `\ufeff` 抛异常被静默吞掉 | 读取用 `utf-8-sig`，写入用 `utf-8` |

### GUI 显示
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 10 | 邮箱字段保存后重启界面仍为空 | `_config_refresh_all` 只刷了 provider/api_key/model，漏了邮箱 Entry | 补充邮箱字段刷新逻辑 |
| 11 | 初始化时模型下拉选项未更新 | `_config_refresh_all` 未在初始化时根据 provider 更新模型选项 | 初始化先更新模型下拉列表 |
| 12 | 测试发送按钮点击无任何反应 | `datetime.now()` 但 datetime 被导入为别名 `_dt`，`NameError` 被 Tkinter 静默吞掉 | 改为 `_dt.datetime.now()` |

### 构建与部署
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 13 | PyInstaller BUILD SUCCESS 但 EXE 不含最新代码 | git CRLF 转换在构建后更新源文件 mtime | 清理 build 缓存 + 强制重编译 |
| 14 | PyInstaller PermissionError 无法覆盖 EXE | 目标 gui.exe 正在运行 | 构建前 Stop-Process，如仍锁定则 rename 旧 EXE |
| 15 | PyInstaller "gui.spec not found" | 错误地在 gui/ 子目录下执行 | spec 在项目根目录 |
| 16 | config 目录被打包进 EXE 且路径错误 | EXE 在 `gui/build/dist/`，CONFIG_DIR=父父目录/config 指向 `gui/build/config` | 构建直接输出到 `E:\stock-tool\dist\gui.exe`，CONFIG_DIR 正确指向 `E:\stock-tool\config` |

### PyInstaller 冻结环境
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 17 | 定期跟踪按钮报 `name 'combo_year_2' is not defined` | 局部变量和 global 声明冲突 | 两处都声明 global，直接访问 |
| 18 | UP 主自动补全名称和保存功能失效 | `_auto_fill_name` 中 `except:pass` 吞掉错误；openpyxl 未打包 | 添加 debug 日志输出；gui.spec hiddenimports 补充 openpyxl/pandas/requests |
| 19 | EXE 启动崩 `ModuleNotFoundError: No module named 'step1_fetch_videos'` | batch_parser.py `import step1_fetch_videos` 未被 PyInstaller 收集 | gui.spec 加 `--paths E:\stock-tool\scripts` 及 hidden-import scripts.step1~5 |
| 20 | EXE 报 `ModuleNotFoundError: No module named 'requests'` | venv 缺 requests，step1 依赖 | pip install requests 到 venv，重建 |
| 21 | cryptography 解密失败，get_setting 返回密文致 DeepSeek 401 | cryptography 未被打包进 EXE | gui.spec hiddenimports 加 cryptography |

### 批量解析
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 22 | 定期跟踪缺少高峰错峰队列 | `button_5_clicked` 无 `is_peak()` 检查 | 高峰弹窗 askyesno，选"否"入队；task_queue_manager 新增 batch_parse 类型 |
| 23 | 单视频/批量重复转写已存在的 txt | 无检查，每次都重新转写 | 检查 txt 非空则返回 `skipped=True` |
| 24 | 无新增转写时仍调用 AI 生成总结 | 跳过视频被标记 success 进入 transcribe_success | 过滤条件新增 `not r.get("skipped")`；new_count==0 时跳过 Phase 2 |
| 25 | 同一天多次批量解析只含当次视频，覆盖前次总结 | 每次只拿当次新转写覆盖写入 | 已有总结时扫描当日全部 txt 合并重新生成 |
| 26 | target_date 设为昨天时目录和总结仍用今天日期 | `date_prefix` 直接取 `datetime.now()` | 改为 effective_date（target_date 存在时解析，否则 now()） |
| 27 | 批量解析跑完但未生成 AI 总结文件 | 之前 EXE 路径错 LLM Key 读不到→转写成功总结失败；再跑转写跳过→new_count==0→跳过总结 | 新增分支：new_count==0 但有存量转写且缺总结时，自动补生成批次总结 |
| 28 | 批次总结文件未写入日期文件夹 | existing_summary 路径取 save_dir 根 | 改为 today_dir / BATCH_SUMMARY_FILENAME |
| 29 | 转换/补摘要后不保存就退出 | 未自动保存 | 在 button_5_clicked 执行完成后自动调用 save_settings |

### 日期选择器
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 30 | 日期 Listbox 被按钮遮挡 | Listbox x=390~636 与按钮 y=504~596 重叠 | 日期标签/Listbox 右移 |
| 31 | 日期选择器改为折叠弹出式 | 设计需求 | 按钮「已选 N 天 ▼」→ Toplevel 浮层，overrideredirect，Ctrl/Shift 多选，Escape 取消 |
| 32 | 按钮太丑且浮层超出窗口 | 尺寸过大位置靠下 | 浅蓝底深蓝字 200×38，y=456；空间不足向上弹出 |
| 33 | 浮层不跟随主窗口移动 | 未绑定 Configure | 绑定主窗口 Configure 实时重定位 |
| 34 | 点击日期反而关闭、无法多选 | overrideredirect Toplevel 事件穿透到主窗口 _on_global_click | grab_set() 方案无效后改用 winfo_toplevel 坐标检测 + after(150) bind_all |
| 35 | 浮层层级不对 | 不应置顶跟随桌面 | 保留 topmost + lift()，FocusOut 误关元凶已移除 |

### 配置页模型
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 36 | DeepSeek API 返回 400 模型无效 | 默认模型 deepseek-chat 已过期 | 默认模型改为 deepseek-v4-pro；settings.json 同步 |
| 37 | 火山方舟下拉仍显示旧模型 | 未更新选项 | 改为 doubao-seed-2-0-lite-260428 / doubao-seed-2-0-mini-260428 |
| 38 | 配置页调试日志无法清空 | 只有浏览按钮 | 新增红色"清空"按钮，点击清除 debug_log 设置 |

### 外观
| # | 问题 | 根因 | 修复 |
|---|------|------|------|
| 39 | 托盘图标绿色不够醒目 | 设计如此 | 背景改为红色 `(178,34,34)` |
| 40 | EXE 无自定义图标 | gui.spec 未指定 icon | 生成 app_icon.ico，gui.spec 添加 icon |
| 41 | 按钮点后变空白 | place_forget() 隐藏 | 改为 `state="disabled", fg="#AAAAAA"` 灰色禁用 |
| 42 | 定期跟踪页表格右侧滚动条多余 | 设计需求 | 移除 ttk.Scrollbar，表格宽 317，保留鼠标滚轮翻页 |

## 已添加功能

**增量解析**：
- `data/parsed_records.json` 维护已解析 BV 号，24h 内跳过转写
- 报告标题带时段名称
- 新增 `backend/parsed_records.py`

**错峰调度**：
- 高峰弹窗确认入队，低谷自动消费
- `backend/task_queue_manager.py`、`backend/time_price_judge.py`、`backend/valley_scheduler.py`

**批量解析日期管理**：
- 目录 `save_dir/mmdd/uid/bvid/`，总结 `save_dir/mmdd/批次总结_YYYY-MM-DD.txt`

**折叠日期选择器**：
- 按钮「已选 N 天 ▼」→ 弹出 Toplevel 多选浮层
- 主窗口移动时跟随，点击外部自动确认，Escape 取消

**配置页**：
- 支持 DeepSeek / 火山方舟切换，模型下拉联动
- 调试日志路径：浏览选择 + 清空按钮

## 关键配置路径

| 配置 | 路径 |
|------|------|
| 配置文件 | `config/settings.json` |
| bili2text 路径 | settings → bili2text_dir |
| B 站 Cookie | settings → cookie_file |
| 调试日志 | settings → debug_log（可清空） |
| LLM 提供商 | settings → llm_provider（deepseek/volcengine） |
| LLM 模型 | settings → llm_model（deepseek-v4-pro/flash 或 doubao-seed-2-0-lite/mini） |
| 已解析记录 | `data/parsed_records.json` |
| 单实例锁 | `%TEMP%\stock_tool_instance.lock` |

## 流水线步骤

`scripts/run_pipeline.py` → step1 拉取 → step2 下载音频 → step3 转写 → step4 提取个股 → step5 分析报告
