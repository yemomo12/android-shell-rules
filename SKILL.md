---
name: android-shell-rules
description: 【本机命令|shell|沙盒|执行报错|adb|跑命令】DevCodex on Android 的执行通道强制规则。任何要跑命令、读写手机文件、处理 JSON、联网抓取的场合，开工前先读本技能。核心：Termux 沙盒已坏，命令一律走 shell_exec（Shizuku ADB）；读写文件优先用 fs_* 工具；大文件禁止用 sed/awk 按行取内容。当命令挂住不返回、报 CANNOT LINK EXECUTABLE、python_run/node_run/pipeline_run 无响应时也读本技能。
---

# 本机执行通道规则（DevCodex on Android）

这份技能只有一个目的：**新对话第一轮就知道命令该走哪条通道**，不用等用户提醒，也不允许先试错再改。

## 铁律

> 所有命令 → `shell_exec`（Shizuku ADB）。
> 所有文件读写 → `fs_*` 工具。
> **沙盒通道一个都不许碰。**

被禁用的工具，不是"先试试能不能用"，是直接不调用：

| 通道 | 状态 | 故障表现 |
|---|---|---|
| `shell_exec`（Shizuku ADB） | ✅ 唯一可用 | uid=2000(shell)，可 ls/cat/cp/curl/tar/sed/awk |
| `linux_exec` | ❌ 坏 | 缺 `libreadline.so.8` |
| `python_run` | ❌ 坏 | `CANNOT LINK EXECUTABLE ... bash` |
| `node_run` | ❌ 坏 | 同上，共用同一 bash |
| `pipeline_run` | ❌ 坏 | 同上 |
| `sandbox_pkg_install` | ❌ 坏 | 装完也起不来，白等 |

## 最坑的地方：坏通道不报错，它挂住

沙盒坏掉的症状**不是**报错，是**不返回**。命令发出去以后没有任何输出，一直挂着，直到调用方超时被打断。

这会伪装成"这条命令很慢"，让人误以为是 `ls` 大目录慢、`python` 加载慢，于是换条命令重试——**还是同一条坏通道，还是挂住**。

所以：**任何命令超过 20 秒没有任何输出，判定为通道问题，不是命令问题。立刻砍掉，改用 `shell_exec`。不要原样重试。**

真实翻车记录（务必引以为戒）：

- 用 `python_run` 去 `json.load` 一个 63KB 的文件 —— 挂住。用户问"怎么卡住了"。
- 改用 `python_run` 查 `os.path.exists` —— 又挂住。用户问"这个命令需要这么久吗，为什么之前不会卡"。
- 两次都白等，一次有效信息都没拿到。同一个沙盒通道，试第二次就是浪费用户时间。

判据很简单：**要跑任何解释器（python/node）→ 本机根本没得跑 → 不走这条。**

## 文件读写：优先 `fs_*`，不绕道 shell

`fs_read` / `fs_write` / `fs_list` / `fs_grep` / `fs_stat` / `fs_find` / `fs_head` / `fs_tail` 走的是应用自身权限，**既不经沙盒也不经 adb**，最稳最快。

只有 `fs_*` 覆盖不到的动作（校验、批量替换、curl、tar）才动 `shell_exec`。

## shell 里千万别做的事

### 1. 不要递归 ls 中文长路径

```sh
# 会卡在 MediaProvider，实测挂住
ls -la "/storage/emulated/0/酒馆破线和正则还有工具/酒馆写卡预设/" ".../写卡正则"
```

`/storage/emulated/0` 走 FUSE，递归 + 中文长路径会卡在媒体扫描上。要看目录内容，用 `fs_list`，或者 `ls` 只列单层、不带 `-R`。

### 2. 不要用 sed/awk 按行读超长行

Android 的 `sed`/`awk` 对**约 4 万字节以上的单行会静默截断**，不报错，结果全错。

已踩过的坑：主题 JSON 的 `custom_css` 字段是一整行 7 万字节，`sed -n '33p'` 读出来被截断，后续切割全错、生成坏文件。

替代做法：

- 字节级切片：`head -c` / `tail -c`
- 取偏移定位：`grep -bo`
- 查控制字符：`od -An -tx1`
- 校验 JSON：先用 awk 扫字符串闭合 + 括号配对，别指望格式化工具

### 3. 不要写进用户文件用 cp

`cp` 会覆盖目标文件原有的属主和权限。写用户目录用重定向：

```sh
cat src > "/storage/emulated/0/目标路径/文件.json"
```

## 本机有什么、没有什么

**有**：toybox 全套（`ls` `cp` `mv` `sed` `awk` `grep` `find` `head` `tail` `wc` `od` `printf`）、`curl` 8.12、`tar`、`base64`。

**没有**：`python`、`python3`、`node`、`jq`、`ruby`、`git`。

**所以**：

- 处理 JSON → 用 awk 状态机 / `grep -c` / 手动切片，别指望 jq
- 跟 GitHub 交互 → 用 `curl` 打 REST API，别指望 `git clone/push`
- 需要真正的 Python 能力 → 如实告诉用户本机没有，别假装

## 路径边界

| 路径 | 可访问 | 说明 |
|---|---|---|
| `/storage/emulated/0/Android/data/ai.devcodex.agent/files/workspace` | ✅ 读写 | 工作区，`fs_*` 的默认位置 |
| `/storage/emulated/0/` 其他目录 | ✅ 读写 | 用户的真实文件都在这 |
| `/data/local/tmp` | ✅ 读写 | 临时文件、header、解包 |
| `/data/user/0/ai.devcodex.agent` | ❌ | 应用私有目录，无 root 进不去 |
| `/system` `/vendor` | ⛔ | 禁止写入 |

## 密钥处理

Token / 密码**不写进工作区文件、不写进仓库、不 echo 到日志**。

`curl` 用临时 header 文件，权限收紧，用完删：

```sh
cd /data/local/tmp && printf 'Authorization: Bearer %s\n' "$TOKEN" > gh.hdr
chmod 600 gh.hdr
curl -s -H @gh.hdr https://api.github.com/user
rm -f gh.hdr
```

## 交付前自检

1. 命令全部走的 `shell_exec` / `fs_*`？有一条走了沙盒就是不合格
2. 有没有哪条命令挂住过？挂住了有没有立刻换通道而不是重试？
3. 改动只落在目标文件，原文件备份或未动？
4. JSON/配置改动过了结构校验，不是"看起来对"？
5. 超长行文件，有没有用行工具断言过内容？（禁止）
6. 哪些没验证，有没有如实说明？
7. 密钥没留在磁盘和对话输出里？

## 如果沙盒修好了

`libreadline.so.8` 补齐、`python_run` 能跑通，就可以恢复用 Python 处理 JSON。届时让用户更新本技能，别自己擅自改回沙盒。
