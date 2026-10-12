# muscriptor 运行时（FuFumidi 修复版）

这里托管 **打了补丁的 `muscriptor` 轮子**，供 FuFumidi 内置/重装转录运行时使用。

## 修的是什么

上游 `muscriptor` 0.3.0 的长音频转录有「僵尸音符」缺陷：**100 分钟的交响录播会产出
长达 4320 秒（全曲 72%）的音符**，波形图上是一片实线墙。

根因（`muscriptor/events.py` 的 `OpenNoteTracker`）：

- 模型按 **5 秒**分块生成，跨块的延音靠每块开头的 **tie 段**维持；
- 块边界只关闭「**不在 tie 段里**」的音符；
- 而 `prelude_forcing` 打开时，tie 段是从「**当前还开着的音符**」强制构造的
  （`transcription_model.py` 里 `tie_section_token_ids(tracker.open_keys())`）。

两条规则叠加 → 「不在 tie 段里」几乎恒为空集 → **块边界永远关不掉任何音符**，
一个音只能靠模型在块内显式吐 note-off 才结束。模型在长音频里漏一次收尾，这个音就被
强制延续下去，形成自我强化。

## 改了什么

在 `OpenNoteTracker` 的 tie 结算处加一条保险：**任何音持续超过 `max_note_hold` 秒就强制
关闭**（关在 `onset + max_note_hold`）。默认 `DEFAULT_MAX_NOTE_HOLD_SEC = 15.0`；
传 `max_note_hold=None` 可关闭，行为与上游 0.3.0 完全一致。

实测（离线，对现有转录结果）：cap=15s 把两个病态文件的最长音从 4320.8s / 2691.7s 压回 15.0s，
而健康文件（时长上限 12.3s）**一个音符都不动**。

## 安装

```bash
python -m pip install --no-deps --upgrade \
  https://raw.githubusercontent.com/monologue82/Models/main/muscriptor-runtime/muscriptor-0.3.0%2Bfufumidi.1-py3-none-any.whl
```

FuFumidi 内置运行时的两处位置（仓库 `resources/python` 与安装版 `resources/python`）
在 0.3.0+fufumidi.1 中已直接带上该修复，无需再装。

## 版本与许可

- 基线：`muscriptor` 0.3.0（PyPI），源码与已装版本逐字节一致
- 本版：`0.3.0+fufumidi.1`（PEP 440 本地版本号，排序高于 0.3.0，不会被 PyPI 版覆盖）
- 许可：**MIT**，Copyright (c) 2026 Kyutai x Mirelo —— 原 LICENSE 已随包保留
- 改动仅限 `muscriptor/events.py` 一个文件

## 校验

```
SHA-256  11828351ae24f22024f7a16f5f093a4192f3826165f459a4fcc134ac7224bae6
文件名   muscriptor-0.3.0+fufumidi.1-py3-none-any.whl
大小     1667587 B
```
