# Claude Code 对话自动总结配置指南

> **怎么用**：把这份文档整份交给 Claude Code，说「按这个装」。它会先问你想把总结存哪个文件夹，然后自动装好并验证。
>
> 装完之后，每次上下文压缩或退出会话时，都会自动生成一份对话摘要。
>
> **适用环境**：Windows / macOS / Linux 上的 Claude Code，Node.js ≥ 18。

---

## 0. 执行者须知

**你（Claude）正在读一份待执行的安装文档。请按顺序执行，不要跳步。**

三条硬性要求：

1. **第一步必须问用户目录，问完再动手。** 不要自己猜、不要用默认值直接装。用 `AskUserQuestion`；如果没有这个工具，就用文字提问，然后**停下等用户回答**。
2. **绝对不要整体覆盖 `settings.json`。** 那个文件里可能有用户的模型配置、环境变量、权限白名单，覆盖会造成严重损失。必须读出来 → 备份 → 合并 → 写回。
3. **装完必须实测一次，并把结果如实报告。** 没跑通就说没跑通，不要只说「已配置完成」。

---

## 1. 第一步：询问保存目录

用 `AskUserQuestion`，问题大意：

> 对话总结的 Markdown 文件想保存在哪个文件夹？

选项按平台给：

| 选项 | 说明 |
|---|---|
| `~/Claude_Dialogue_Summary`（推荐） | 放在用户主目录下，所有项目共用一个总结库 |
| `<当前项目>/.claude-summaries` | 每个项目单独一个库，跟随项目走 |
| 其他 | 用户自己填绝对路径 |

用户选完后，把路径规范化成**绝对路径**并展开 `~`。如果目录不存在，创建它。

拿到路径后，**把它记为 `OUTPUT_DIR`**，后面所有步骤都用到它。

### 必须确认的额外信息

顺便检查这两项，缺哪个问哪个：

- **`node` 是否可用**（脚本是 Node.js 写的）：执行 `node -v`，要求 **≥ 18**
- **`claude` 可执行文件在哪**：见第 2 步

---

## 2. 环境预检

依次执行，任何一项不满足就停下来告诉用户：

```bash
node -v                  # 需要 ≥ 18
where claude             # Windows；macOS/Linux 用 which claude
```

找 `claude` 时按这个顺序试，记住找到了哪个：

1. `~/.local/bin/claude.exe`（Windows 原生安装）
2. `~/.local/bin/claude`（macOS/Linux 原生安装）
3. `~/AppData/Roaming/npm/claude.cmd`（Windows npm 安装）
4. `/usr/local/bin/claude`、`/opt/homebrew/bin/claude`
5. 系统 `PATH` 里的 `claude`

**一个都找不到就先停下**：让用户先装好 Claude Code CLI，否则脚本无法调用模型。

---

## 3. 写文件

一共四个文件，都是新建或合并，不会覆盖用户已有内容（`settings.json` 除外，见下）。

### 3.1 `~/.claude/hooks/summarize.js`

把本文档**附录 A** 的代码原样写入。**不需要改任何内容** —— 目录配置走独立文件，脚本本身不含机器相关路径。

如果附录 A 的版本和用户已有的同名文件冲突（重复安装），先备份：
`summarize.js.bak-<时间戳>`。

### 3.2 `~/.claude/hooks/summary-config.json`

这是**唯一需要按用户情况写**的文件：

```json
{
  "outputDir": "<第 1 步拿到的绝对路径，反斜杠写成 \\\\>",
  "model": "haiku",
  "claudePath": "<可选：第 2 步找到的 claude 路径>",
  "deleteWithTranscript": true
}
```

Windows 示例（路径里的 `\` 在 JSON 里必须转义成 `\\`）：

```json
{
  "outputDir": "D:\\<你的目录>\\Claude_Dialogue_Summary",
  "model": "haiku"
}
```

macOS/Linux 示例：

```json
{
  "outputDir": "/Users/alice/Claude_Dialogue_Summary",
  "model": "haiku"
}
```

字段说明：

| 字段 | 必填 | 说明 |
|---|---|---|
| `outputDir` | 是 | 摘要输出目录，不存在会自动创建 |
| `model` | 否 | 默认 `haiku`。用小模型即可，单次成本可忽略 |
| `claudePath` | 否 | 默认自动探测。探测失败时手动指定 |
| `deleteWithTranscript` | 否 | 默认 `false`。开启后，删掉原始对话记录时对应的 `.md` 也一起清理，见第 5 节 |

> **为什么单独放配置文件**：这样别人拿到脚本不用改代码，只改一个 JSON；将来换目录也只改这里。

### 3.3 合并 `~/.claude/settings.json`

**这是最危险的一步。** 严格按下面的流程做，不要手工整体重写：

1. 先读现有 `settings.json`。如果不存在，当作 `{}`
2. 备份成 `settings.json.bak-before-summary`
3. 用下面这段 Node 脚本合并（它会自动去重，重复执行也安全）

把这段存成临时文件 `_install-hooks.js`，用 `node _install-hooks.js` 跑一次，**跑完删除临时文件**：

```javascript
const fs = require('fs');
const path = require('path');
const os = require('os');

const p = path.join(os.homedir(), '.claude', 'settings.json');
const s = fs.existsSync(p)
  ? JSON.parse(fs.readFileSync(p, 'utf8').replace(/^\uFEFF/, ''))
  : {};
if (fs.existsSync(p)) fs.copyFileSync(p, p + '.bak-before-summary');

// 上下文默认只留 30 天，会被自动删掉，摘要里的原始记录链接就废了
s.cleanupPeriodDays = Math.max(Number(s.cleanupPeriodDays) || 0, 3650);

const cmd = 'node "' + path.join(os.homedir(), '.claude', 'hooks', 'summarize.js') + '"';
s.hooks = s.hooks || {};
// SessionStart 负责开场清理孤儿摘要（见第 6 节），它不产生摘要，只是顺手扫一遍
for (const ev of ['PreCompact', 'SessionEnd', 'SessionStart']) {
  s.hooks[ev] = s.hooks[ev] || [];
  // 已经装过就不重复添加
  const has = s.hooks[ev].some(g =>
    (g.hooks || []).some(h => String(h.command || '').includes('summarize.js')));
  if (!has) s.hooks[ev].push({ hooks: [{ type: 'command', command: cmd, timeout: 10 }] });
}

fs.writeFileSync(p, JSON.stringify(s, null, 2) + '\n', 'utf8');
console.log('hooks merged:', Object.keys(s.hooks).join(', '));
```

合并后 `settings.json` 里会多出：

```json
{
  "cleanupPeriodDays": 3650,
  "hooks": {
    "PreCompact": [
      { "hooks": [{ "type": "command", "command": "node \"C:\\Users\\<你>\\.claude\\hooks\\summarize.js\"", "timeout": 10 }] }
    ],
    "SessionEnd": [
      { "hooks": [{ "type": "command", "command": "node \"C:\\Users\\<你>\\.claude\\hooks\\summarize.js\"", "timeout": 10 }] }
    ],
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "node \"C:\\Users\\<你>\\.claude\\hooks\\summarize.js\"", "timeout": 10 }] }
    ]
  }
}
```

**跑完必须复查**：读一遍写好的 `settings.json`，确认原有的 `env`、`permissions`、`model` 等字段都还在。少了任何一项，立刻用刚备份的 `settings.json.bak-before-summary` 恢复并报告问题。

> `timeout: 10` 是给**派发**用的，不是给总结用的。派发瞬间返回（<1 秒），真正的总结在后台子进程里跑。

### 3.4 `~/.claude/commands/summary.md`

写入下面内容，但**必须把 `<SUMMARIZE_JS>` 替换成第 2 步确认过的脚本绝对路径**（`~` 在 Windows 的 node 里不会被展开，写死路径最稳）：

- Windows：`C:\Users\<用户名>\.claude\hooks\summarize.js`
- macOS/Linux：`/Users/<用户名>/.claude/hooks/summarize.js`

```markdown
---
description: 立即对当前对话生成一份总结，写入配置的总结目录
---

运行下面的命令，为当前项目最近一次对话生成总结，然后把命令输出原样转述给用户（不要改写成你自己的话，也不要省略）：

```
node "<SUMMARIZE_JS>" --manual
```

如果输出是"没有新内容需要总结"，就告诉用户距上次总结后新增内容太少，继续聊一会儿再试。
如果输出里包含 `已写入：`，把路径那一行和总结正文一起展示出来。
```

> **注意**：`allowed-tools` 不是合法的 frontmatter 字段，加了会被忽略，不要写。

---

## 4. 验证

按顺序做，每步都要真跑，不要只是「检查了一下应该没问题」。

### 4.1 语法检查

```bash
node --check ~/.claude/hooks/summarize.js
```

### 4.2 实跑一次

在用户的项目目录下执行：

```bash
node ~/.claude/hooks/summarize.js --manual
```

预期三种结果：

| 输出 | 含义 | 处理 |
|---|---|---|
| `已写入：<路径>` | 成功 | 继续 4.3 |
| `没有新内容需要总结（…新增内容太少）` | 增量逻辑正常，本次没东西可写 | **也算成功**，说明链路通了 |
| `没找到对话记录：<路径>` | 当前目录不是 Claude Code 项目，或还没产生对话 | 让用户在本项目里聊几句再试 |

如果输出为空或者报错，看日志：

```bash
type %USERPROFILE%\.claude\hooks\summary-state\summarize.log
# macOS/Linux: cat ~/.claude/hooks/summary-state/summarize.log
```

**日志完全为空 = hook 或 stdin 解析失败**，最常见原因是 `settings.json` 格式错，或 Windows 管道带 UTF-8 BOM（附录 A 的代码已处理 BOM）。

### 4.3 检查产物

确认 `OUTPUT_DIR` 下出现了：

```
<开始日期>_<两级项目名>_<标题>_<8位ID>.md
INDEX.md
```

打开 md，逐项核对：

- [ ] 文件头「原始记录」的路径真实存在
- [ ] 「恢复命令」里的 `claude --resume <sid>` 能恢复对话
- [ ] 「涉及文件」里没有总结目录自身的文件
- [ ] 「概览」写了 2-4 句
- [ ] 每次总结段落含「做了什么」「结论」，标题和关键词齐全

### 4.4 验证增量

**再跑一次** `--manual`。这次应该输出「没有新内容需要总结」或「新增内容太少」—— 说明不会把同一段内容重复总结。

### 4.5 验证自动触发

让用户**退出 Claude Code**（`/exit` 或连按两次 `Ctrl+C`），等 30 秒，然后看日志末尾是否有：

```
收到 SessionEnd（退出时）session=...
已写入 <路径>
```

有就说明自动触发装好了。

---

## 5. 可选：删除原始记录时，连带清理摘要

### 5.1 先问用户要不要开

装完主功能后，问一次：

> 你删掉某个对话的原始记录后，对应的 `.md` 摘要要怎么处理？
>
> - **一起删掉**：保持一一对应，删了对话就不留摘要
> - **保留**：摘要当独立档案存着，即使原始记录没了

用户选「一起删掉」，就在 `summary-config.json` 里写 `"deleteWithTranscript": true`；选「保留」就不写这个字段（默认 `false`）。

**没有明确答复时不要开启**，装好总结功能即可。

### 5.2 它怎么工作

Claude Code 的 `SessionStart` 钩子会在**每次打开 Claude Code 时**触发。脚本收到后起一个后台进程扫一遍 `~/.claude/hooks/summary-state/`：

| 检查 | 处理 |
|---|---|
| 原始 `.jsonl` 还在 | 不动 |
| 原始 `.jsonl` 没了，且摘要归属可确认 | 删 `.md`、删 state、从 `INDEX.md` 去掉那一行 |
| 原始 `.jsonl` 没了，但归属确认不了 | 跳过，不删 |
| 一条 `.jsonl` 都扫不到 | 整体放弃，防止环境不对时清空整个目录 |

**为什么挂在 `SessionStart` 而不是定时任务**：Claude Code 没有常驻进程，装个定时任务反而多一份要维护的东西。挂在开场，你下次用它的时候顺手就清了，不需要额外机制。

**没有等待期。** 删了就是删了 —— 这是你明确选择的语义。如果你担心误删，别开这个开关，用下面的预览命令。

### 5.3 它不会删什么

三道限制，都在代码里：

1. **只删 state 里登记过的文件**。state 里的路径必须落在 `OUTPUT_DIR` 里、必须是 `.md`。所以你在总结目录里放的其他笔记（比如这份指南本身）不会被碰。
2. **摘要正文必须含 `- 对话 ID：<完整UUID>`**。这是文件头里写死的，改不了名就对不上，对不上就不删。
3. **扫描失败即放弃**。`~/.claude/projects` 读不到、或一条 jsonl 都没有，直接返回不清理。

还有一类情况：**没有 state 文件的老摘要**（比如早年手动整理的）。脚本不按文件名猜，只会在输出里提示一句「另有 N 个文件没有登记状态，未处理」。

### 5.4 手动执行

```bash
# 预览：列出会删哪些，不删任何东西
node ~/.claude/hooks/summarize.js --sweep

# 真删
node ~/.claude/hooks/summarize.js --sweep --yes
```

预览和实删走同一套判断，预览看到什么，`--yes` 就删什么。

### 5.5 验证

先造一条孤儿再验，别拿真实对话做实验：

1. 记下一个已总结对话的 `session_id`（摘要文件头里的「对话 ID」）
2. 把 `~/.claude/projects/<项目目录>/<session_id>.jsonl` **移到别的地方**（不要直接删）
3. 运行 `--sweep`，应该看到 `将删除：<文件名>`
4. 运行 `--sweep --yes`，确认 `.md` 和 `INDEX.md` 那行都没了
5. **把 jsonl 移回原位**，再跑一次 `--sweep`，应该看到「没有孤儿总结」

第 5 步很重要：它证明脚本认的是「原始记录在不在」，不是「删过一次就永久记账」。

### 5.6 关掉它

把 `summary-config.json` 里的 `deleteWithTranscript` 改成 `false`，或者直接删掉这个字段。已经删掉的摘要不会恢复，但之后不会再清。

`SessionStart` 钩子留着也无害 —— 开关为 `false` 时脚本压根不会启动清理进程。

---

## 6. 收尾并报告

清理临时文件（`_install-hooks.js` 等），然后按这个格式向用户汇报：

```
✅ 对话自动总结已装好

- 保存目录：<OUTPUT_DIR>
- 触发时机：上下文压缩前 / 会话退出时 / 手动 /summary
- 连带清理：<已开启（删对话时同步删摘要） / 未开启>
- 配置文件：~/.claude/hooks/summary-config.json
- 日志：~/.claude/hooks/summary-state/summarize.log
- settings.json 备份：~/.claude/settings.json.bak-before-summary

实测结果：<已写入 xxx.md / 无新内容（增量正常）>

⚠️ 需要重启 Claude Code 一次，让 cleanupPeriodDays 和新钩子生效
   （否则对话记录默认 30 天会被自动清理，摘要里的原始记录链接会失效）
```

如果 `/summary` 命令重启后仍不出现，提示用户再重启一次 —— 斜杠命令在启动时加载。

---

## 附录 A：完整脚本 `summarize.js`

```javascript
// 对话自动总结：PreCompact / SessionEnd hook 触发，后台调用 claude -p 生成凝练总结
const fs = require('fs');
const path = require('path');
const os = require('os');
const { spawn, spawnSync } = require('child_process');

// 机器相关的设置放在同目录的 summary-config.json，代码本身不用改
const CONFIG_FILE = path.join(__dirname, 'summary-config.json');
let CONFIG = {};
try { CONFIG = JSON.parse(fs.readFileSync(CONFIG_FILE, 'utf8').replace(/^﻿/, '')); } catch {}
const SUMMARY_DIR = CONFIG.outputDir || '';
const STATE_DIR = path.join(os.homedir(), '.claude', 'hooks', 'summary-state');
const LOG_FILE = path.join(STATE_DIR, 'summarize.log');
const MODEL = CONFIG.model || 'haiku';
// 原始记录没了（手动删文件，或在会话列表里删对话）时，是否连带删掉对应总结
// 默认关闭：cleanupPeriodDays 到期时系统也会删原始记录，那种情况下不该跟着删总结
const DELETE_WITH_TRANSCRIPT = CONFIG.deleteWithTranscript === true;
const MAX_CHARS = 60000;   // 发给模型的对话文本上限，超出保留开头和结尾
const MIN_CHARS = 80;      // 新增内容太少就不总结
const GUARD = 'CC_SUMMARY_CHILD';

function log(msg) {
  try {
    fs.mkdirSync(STATE_DIR, { recursive: true });
    fs.appendFileSync(LOG_FILE, `[${new Date().toISOString()}] ${msg}\n`, 'utf8');
  } catch {}
}

function readStdin() {
  try { return fs.readFileSync(0, 'utf8').replace(/^﻿/, ''); } catch { return ''; }
}

function main() {
  if (process.env[GUARD]) return;               // 总结用的子进程自己也会触发 hook，直接跳过
  const args = process.argv.slice(2);
  // 配置缺失时不给明确提示的话，hook 会静默失败，很难排查
  if (!SUMMARY_DIR) {
    log(`配置缺失：${CONFIG_FILE} 里没有 outputDir`);
    if (args[0] === '--manual') console.log(`没有配置保存目录：请在 ${CONFIG_FILE} 里填写 outputDir（绝对路径）`);
    return;
  }
  if (args[0] === '--worker') return worker(JSON.parse(Buffer.from(args[1], 'base64').toString('utf8')));
  if (args[0] === '--manual') return manual(args[1] || process.cwd());
  if (args[0] === '--sweep') {
    // 不带 --yes 只预览；带 --yes 才真删
    console.log(sweepOrphans(!args.includes('--yes')));
    return;
  }
  if (args[0] === '--sweep-run') {
    try { const r = sweepOrphans(false); log(`开场清理：${r}`); } catch (e) { log(`开场清理失败：${e.message}`); }
    return;
  }

  let input = {};
  try { input = JSON.parse(readStdin() || '{}'); } catch (e) { log(`stdin 解析失败: ${e}`); return; }
  const ev = input.hook_event_name;

  // 开场顺手清一遍孤儿总结：原始记录已不存在的，对应总结也删掉
  // 这一步不依赖 transcript_path，所以放在下面的检查之前；后台跑，不拖慢启动
  if (DELETE_WITH_TRANSCRIPT && ev === 'SessionStart') {
    const sweep = spawn(process.execPath, [__filename, '--sweep-run'],
      { detached: true, stdio: 'ignore', windowsHide: true });
    sweep.unref();
  }

  if (!input.transcript_path) { log(`hook 缺少 transcript_path: ${JSON.stringify(input).slice(0, 200)}`); return; }
  const REASONS = { prompt_input_exit: '退出时', logout: '登出时', clear: '/clear 时', other: '会话结束时' };
  const trigger = ev === 'PreCompact'
    ? (input.trigger === 'manual' ? '手动压缩前' : '自动压缩前')
    : (REASONS[input.reason] || '退出时');
  const job = { sessionId: input.session_id, transcript: input.transcript_path, cwd: input.cwd, trigger };
  log(`收到 ${ev}（${trigger}）session=${input.session_id}，转后台处理`);
  // 后台运行，不阻塞压缩和退出
  const child = spawn(process.execPath, [__filename, '--worker', Buffer.from(JSON.stringify(job)).toString('base64')],
    { detached: true, stdio: 'ignore', windowsHide: true });
  child.unref();
}

// /summary 命令：找当前项目最近修改的 jsonl，同步总结并打印结果
function manual(cwd) {
  const dir = path.join(os.homedir(), '.claude', 'projects', cwd.replace(/[^A-Za-z0-9]/g, '-'));
  let files = [];
  try { files = fs.readdirSync(dir).filter(f => f.endsWith('.jsonl')); } catch {}
  if (!files.length) { console.log(`没找到对话记录：${dir}`); return; }
  files.sort((a, b) => fs.statSync(path.join(dir, b)).mtimeMs - fs.statSync(path.join(dir, a)).mtimeMs);
  const job = { sessionId: files[0].replace(/\.jsonl$/, ''), transcript: path.join(dir, files[0]), cwd, trigger: '手动总结' };
  const out = worker(job);
  console.log(out || '没有新内容需要总结（距上次总结后新增内容太少）。');
}

function worker(job) {
  fs.mkdirSync(STATE_DIR, { recursive: true });
  const lock = path.join(STATE_DIR, `${job.sessionId}.lock`);
  if (!acquireLock(lock)) { log(`锁超时，跳过 ${job.sessionId}`); return null; }
  try {
    return summarize(job);
  } catch (e) {
    log(`失败 ${job.sessionId}: ${e && e.stack || e}`);
    return null;
  } finally {
    try { fs.unlinkSync(lock); } catch {}
  }
}

// 同一对话可能同时触发两次（压缩 + 退出），用锁文件排队
function acquireLock(lock) {
  for (let i = 0; i < 120; i++) {
    try { fs.writeFileSync(lock, String(process.pid), { flag: 'wx' }); return true; }
    catch {
      try { if (Date.now() - fs.statSync(lock).mtimeMs > 10 * 60 * 1000) fs.unlinkSync(lock); } catch {}
      Atomics.wait(new Int32Array(new SharedArrayBuffer(4)), 0, 0, 1000);
    }
  }
  return false;
}

function summarize(job) {
  // 会话可能一条消息都没发就退出，jsonl 根本没生成，这种情况直接跳过
  if (!job.transcript || !fs.existsSync(job.transcript)) {
    log(`对话文件不存在，跳过 ${job.sessionId}`);
    return null;
  }
  const stateFile = path.join(STATE_DIR, `${job.sessionId}.json`);
  let state = { lines: 0 };
  try { state = JSON.parse(fs.readFileSync(stateFile, 'utf8')); } catch {}

  const t = parseTranscript(job.transcript, state.lines || 0);
  let body = t.messages.join('\n\n');
  if (body.length < MIN_CHARS) { log(`新增内容太少，跳过 ${job.sessionId}`); return null; }
  if (body.length > MAX_CHARS) {
    body = body.slice(0, MAX_CHARS / 2) + '\n\n……（中间省略）……\n\n' + body.slice(-MAX_CHARS / 2);
  }

  // 先读旧文件（拿概览和旧文件列表），再调模型
  const oldFile = state.file && fs.existsSync(state.file) ? state.file : null;
  const parsed = oldFile ? parseMd(fs.readFileSync(oldFile, 'utf8')) : { sections: [] };
  const overviewOld = blockBody(parsed.sections, '概览');
  const filesOld = blockBody(parsed.sections, '涉及文件')
    .split('\n').map(s => s.replace(/^-\s*/, '').trim()).filter(Boolean);
  const bodySections = parsed.sections.filter(s => s.title !== '概览' && s.title !== '涉及文件');

  const out = callClaude(body, overviewOld, state.title);
  if (!out) return null;

  // 文件名：日期 + 标题 + 8 位 ID。标题由模型随对话演进重写，改名时同步更新
  const titleNow = out.title && out.title !== '无' ? out.title : (t.title || '未命名对话');
  const proj = t.cwd || job.cwd;
  let file = oldFile || path.join(SUMMARY_DIR, buildFileName(t, job.sessionId, titleNow, proj));
  if (oldFile) {
    const wanted = path.join(path.dirname(oldFile), buildFileName(t, job.sessionId, titleNow, proj));
    // 目标名已被别的文件占用时保持原名，不覆盖
    if (wanted !== oldFile && !fs.existsSync(wanted)) {
      try { fs.renameSync(oldFile, wanted); file = wanted; log(`改名 ${path.basename(oldFile)} -> ${path.basename(wanted)}`); }
      catch (e) { log(`改名失败: ${e.message}`); }
    }
  }

  const when = fmtTime(new Date());
  // 决策留空就不写这一行；关键词始终占一行
  const lines = [`**做了什么**：${out.did}`, `**结论**：${out.concl}`];
  if (out.decision && out.decision !== '无') lines.push(`**决策**：${out.decision}`);
  if (out.keywords) lines.push(`**关键词**：${out.keywords}`);
  const section = lines.join('\n');
  bodySections.push({ title: `${when}（${job.trigger}）`, body: `\n${section}\n` });

  // 涉及文件：本次新增的 + 之前记录的，去掉总结目录自身的文件（含旧版本残留）
  const inSummaryDir = p => path.resolve(p).startsWith(path.resolve(SUMMARY_DIR));
  const files = [...new Set([...filesOld, ...t.files])].filter(p => !inSummaryDir(p)).slice(0, 60);
  const overview = out.overview || overviewOld;
  // 文件头每次重建，旧格式的总结文件会顺带升级
  writeSummary(file, buildHeader(t, job, titleNow), overview, files, bodySections);

  fs.writeFileSync(stateFile, JSON.stringify({ lines: t.totalLines, file, title: titleNow, transcript: job.transcript }, null, 2), 'utf8');
  updateIndex({
    sessionId: job.sessionId, title: titleNow, cwd: t.cwd || job.cwd,
    date: when.slice(0, 10), overview, fileName: path.basename(file),
  });
  log(`已写入 ${file}`);
  return `已写入：${file}\n\n## ${when}（${job.trigger}）\n${section}\n`;
}

// 把 md 拆成「标题下方的文件头」+ 各个 ## 段落，便于重排
function parseMd(content) {
  const idx = content.search(/^## /m);
  if (idx < 0) return { header: content.trimEnd(), sections: [] };
  const sections = content.slice(idx).split(/^## /m).slice(1).map(chunk => {
    const nl = chunk.indexOf('\n');
    return nl < 0 ? { title: chunk.trim(), body: '' }
                  : { title: chunk.slice(0, nl).trim(), body: chunk.slice(nl + 1) };
  });
  return { header: content.slice(0, idx).trimEnd(), sections };
}

function blockBody(sections, title) {
  const s = sections.find(x => x.title === title);
  return s ? s.body.trim() : '';
}

// 固定顺序重排：文件头 → 概览 → 涉及文件 → 各次总结
function writeSummary(file, header, overview, files, sections) {
  const parts = [header.trimEnd()];
  if (overview) parts.push(`## 概览\n\n${overview.trim()}`);
  if (files.length) parts.push(`## 涉及文件\n\n${files.map(f => `- ${f}`).join('\n')}`);
  for (const s of sections) parts.push(`## ${s.title}\n${s.body.trimEnd()}`);
  fs.mkdirSync(SUMMARY_DIR, { recursive: true });
  fs.writeFileSync(file, parts.join('\n\n') + '\n', 'utf8');
}

// 所有对话的索引，一行一个，便于整体浏览和交给其他 AI
function updateIndex(info) {
  const idx = path.join(SUMMARY_DIR, 'INDEX.md');
  const line = `- ${info.date} · **${info.title || '未命名对话'}** · ${projectTag(info.cwd)} · ` +
    `${(info.overview || '').replace(/\s+/g, ' ').slice(0, 90)} · [${info.fileName}](${encodeURI(info.fileName)}) ` +
    `<!-- sid:${info.sessionId.slice(0, 8)} -->`;
  let content = '';
  try { content = fs.readFileSync(idx, 'utf8'); } catch {}
  const sid = `<!-- sid:${info.sessionId.slice(0, 8)} -->`;
  const lines = content.split('\n').filter(l => !l.includes(sid));
  const items = lines.filter(l => l.startsWith('- '));
  const head = lines.filter(l => !l.startsWith('- '));
  items.push(line);
  const headText = head.join('\n').trim() || '# 对话总结索引';
  fs.writeFileSync(idx, `${headText}\n\n${items.join('\n')}\n`, 'utf8');
}

// 项目路径取两级：父目录 / 项目名，例如 cc / EverydayConversation
function projectTag(cwd) {
  const parts = String(cwd || '').split(/[\\/]/).filter(Boolean);
  return parts.slice(-2).join(' / ') || '未知项目';
}

// ---- 孤儿总结清理 ----
// 原始记录已不存在的总结就是孤儿：你在会话列表里删了对话，或者手动删了那个 jsonl。
// 做法很轻：每次开 Claude Code 时后台扫一遍，扫到孤儿就清掉。
// fail closed —— 扫描失败、归属对不上，都当作「不确定」，宁可不删。

const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

// 列出所有还活着的会话 ID。读不到就抛错，绝不当成「全都没了」
function inventoryTranscripts() {
  const root = path.join(os.homedir(), '.claude', 'projects');
  if (!fs.lstatSync(root).isDirectory()) throw new Error(`${root} 不是目录`);
  const ids = new Set();
  for (const d of fs.readdirSync(root)) {
    const dir = path.join(root, d);
    let st;
    try { st = fs.lstatSync(dir); } catch { continue; }
    if (!st.isDirectory()) continue;
    let files = [];
    try { files = fs.readdirSync(dir); } catch { continue; }
    for (const f of files) if (f.endsWith('.jsonl')) ids.add(f.slice(0, -6).toLowerCase());
  }
  return ids;
}

// 只删登记在案、且正文写死了完整会话 ID 的 md。
// 这样总结目录里你手写的笔记（比如那份配置指南）永远不会被碰到
function isOwnedSummary(md, sessionId) {
  if (!md || !fs.existsSync(md)) return false;
  if (path.dirname(path.resolve(md)) !== path.resolve(SUMMARY_DIR)) return false;
  if (path.extname(md) !== '.md') return false;
  try { return fs.readFileSync(md, 'utf8').includes(`- 对话 ID：${sessionId}`); }
  catch { return false; }
}

// preview=true 只报告不删
function sweepOrphans(preview) {
  let alive;
  try { alive = inventoryTranscripts(); }
  catch (e) { return `读不到原始记录目录，本次不清理：${e.message}`; }
  if (!alive.size) return '一条原始记录都没找到，为避免误删，本次不清理';

  let states = [];
  try { states = fs.readdirSync(STATE_DIR).filter(f => f.endsWith('.json')); } catch {}

  const out = [];
  const seen = new Set();

  for (const name of states) {
    const sessionId = name.slice(0, -5);
    if (!UUID.test(sessionId)) continue;          // 跳过非会话文件
    seen.add(sessionId);

    let state = {};
    try { state = JSON.parse(fs.readFileSync(path.join(STATE_DIR, name), 'utf8')); } catch { continue; }

    // 原始记录还在，什么都不做
    if (alive.has(sessionId.toLowerCase()) || (state.transcript && fs.existsSync(state.transcript))) continue;

    // 归属校验不过就不动它，宁可漏删也不误删
    if (!isOwnedSummary(state.file, sessionId)) {
      out.push(`跳过（无法确认归属）：${state.title || sessionId.slice(0, 8)}`);
      continue;
    }

    const md = state.file;
    if (preview) { out.push(`将删除：${path.basename(md)}`); continue; }

    try {
      fs.unlinkSync(md);
      try { fs.unlinkSync(path.join(STATE_DIR, name)); } catch {}
      try { fs.unlinkSync(path.join(STATE_DIR, `${sessionId}.lock`)); } catch {}
      dropIndex(sessionId);
      log(`清理孤儿总结 session=${sessionId} ${path.basename(md)}`);
      out.push(`已删除：${path.basename(md)}`);
    } catch (e) {
      out.push(`删除失败：${path.basename(md)} ${e.message}`);
    }
  }

  // 没有登记状态的老总结不靠文件名猜，只提示
  let legacy = [];
  try {
    const known = new Set([...seen, ...alive].map(id => id.slice(0, 8).toLowerCase()));
    legacy = fs.readdirSync(SUMMARY_DIR)
      .filter(f => /_[0-9a-f]{8}\.md$/.test(f) && !known.has(f.slice(-11, -3).toLowerCase()));
  } catch {}

  if (!out.length) {
    return legacy.length
      ? `没有可清理的总结。另有 ${legacy.length} 个文件没有登记状态，无法确认归属，未处理。`
      : '没有孤儿总结，原始记录都还在';
  }
  if (legacy.length) out.push('', `另有 ${legacy.length} 个没有登记状态的 md，无法确认归属，未处理。`);
  if (preview) out.push('', '以上是预览，未删除任何文件。加 --yes 才真正删除。');
  return out.join('\n');
}

// 从 INDEX.md 里去掉某个对话那一行
function dropIndex(sessionId) {
  const idx = path.join(SUMMARY_DIR, 'INDEX.md');
  let content = '';
  try { content = fs.readFileSync(idx, 'utf8'); } catch { return; }
  const sid = `<!-- sid:${sessionId.slice(0, 8)} -->`;
  const kept = content.split('\n').filter(l => !l.includes(sid));
  if (kept.length === content.split('\n').length) return;
  fs.writeFileSync(idx, kept.join('\n'), 'utf8');
}

// 只取用户和助手的文字内容；fromLine 之前的行已总结过
function parseTranscript(file, fromLine) {
  const raw = fs.readFileSync(file, 'utf8');
  const lines = raw.split('\n');
  if (!raw.endsWith('\n')) lines.pop();         // 最后一行可能还没写完
  const r = { title: '', firstUser: '', firstTs: '', cwd: '', messages: [], files: [], totalLines: lines.length };
  lines.forEach((line, i) => {
    let o; try { o = JSON.parse(line); } catch { return; }
    if (o.type === 'ai-title' && o.aiTitle) r.title = o.aiTitle;
    if (o.type !== 'user' && o.type !== 'assistant') return;
    if (o.isSidechain || o.isMeta || !o.message) return;
    if (!r.firstTs && o.timestamp) r.firstTs = o.timestamp;
    if (!r.cwd && o.cwd) r.cwd = o.cwd;
    const text = extractText(o.message.content);
    // 记录本次新增内容里碰过的文件，供文件头「涉及文件」汇总
    if (i >= fromLine && o.message.tool_use_result === undefined && Array.isArray(o.message.content)) {
      for (const b of o.message.content) {
        if (!b || b.type !== 'tool_use' || !b.input) continue;
        const p = b.input.file_path || b.input.path || b.input.notebook_path;
        if (p && typeof p === 'string') r.files.push(p);
      }
    }
    if (!text) return;
    if (o.type === 'user' && !r.firstUser) r.firstUser = text;
    if (i >= fromLine) r.messages.push(`【${o.type === 'user' ? '用户' : '助手'}】${text}`);
  });
  return r;
}

function extractText(content) {
  let s = typeof content === 'string' ? content
    : Array.isArray(content) ? content.filter(b => b && b.type === 'text').map(b => b.text).join('\n') : '';
  s = s.replace(/<(system-reminder|local-command-caveat|local-command-stdout|command-name|command-message|command-args)>[\s\S]*?<\/\1>/g, '')
  return s.trim();
}

// 日期_项目_标题_8位ID.md；项目取两级并去掉分隔斜杠，便于跨项目浏览时一眼看出归属
function buildFileName(t, sessionId, title, cwd) {
  const d = t.firstTs ? new Date(t.firstTs) : new Date();
  const date = fmtTime(d).slice(0, 10);
  const proj = projectTag(cwd || t.cwd).replace(/\s*\/\s*/g, '-')
    .replace(/[\\:*?"<>|\r\n\t]/g, ' ').replace(/\s+/g, ' ').trim();
  const name = (title || t.title || t.firstUser.slice(0, 20) || '未命名对话')
    .replace(/[\\/:*?"<>|\r\n\t]/g, ' ').replace(/\s+/g, ' ').trim().slice(0, 50);
  return `${date}_${proj}_${name}_${sessionId.slice(0, 8)}.md`;
}

function buildHeader(t, job, title) {
  const cwd = t.cwd || job.cwd || '';
  const dir = path.dirname(job.transcript).replace(os.homedir(), '~');
  return [
    `# ${title || t.title || '未命名对话'}`,
    '',
    `- 项目：${projectTag(cwd)}`,
    `- 对话 ID：${job.sessionId}`,
    `- 原始记录：${dir}${path.sep}${path.basename(job.transcript)}`,
    `- 开始时间：${t.firstTs ? fmtTime(new Date(t.firstTs)) : '未知'}`,
    `- 恢复命令：cd "${cwd}"; claude --resume ${job.sessionId}`,
    '',
  ].join('\n');
}

function fmtTime(d) {
  const p = n => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())} ${p(d.getHours())}:${p(d.getMinutes())}`;
}

const PROMPT = (body, overviewOld, scale, oldTitle) => `下面是一段用户与 AI 编程助手的对话记录。请用中文总结，严格按下面的格式输出，不要有额外文字。

篇幅按内容多少来定：${scale}。

【标题】用不超过 15 个字概括这个对话的主题，要能一眼看懂在做什么。${oldTitle
  ? `对话此前的标题是「${oldTitle}」。如果主题仍是同一件事，必须原样输出这个标题，不要改写；只有主题确实发生了明显转变时才换新的。`
  : '宁可是对话的主题，也不要照抄第一句话。'}
【做了什么】说清这段新增内容主要做了哪些事。
【结论】说清得出什么结论、产出什么文件或代码；没有明确结论就写"无"。
【决策】只写这段里做出的关键取舍及理由；没有就写"无"。
【关键词】5-8 个词，用顿号分隔，覆盖主题、技术名词、文件名，便于日后检索。
【概览】${overviewOld
  ? `下面是已有的整体概览，结合这段新增内容改写为覆盖整个对话的新概览，保持简练（一般 2-3 句，对话很长可到 4 句）。只输出概览正文：\n（旧概览）${overviewOld}`
  : '2-3 句话概括整个对话要解决什么问题、进展到哪。'}

对话记录：
${body}`;

// 按新增内容的多少决定篇幅，避免短对话写太长、长对话写不全
function lengthScale(chars) {
  if (chars < 1500) return '这段内容不多，每项 1 句即可，不要展开';
  if (chars < 8000) return '每项 1-2 句';
  if (chars < 30000) return '每项 2-3 句，可具体些';
  return '每项 3-5 句，这段内容很多，要把要点写全';
}

// 各平台、各安装方式的 claude 位置都试一遍；CONFIG.claudePath 可手动指定
function claudeExe() {
  if (CONFIG.claudePath && fs.existsSync(CONFIG.claudePath)) return CONFIG.claudePath;
  const candidates = [
    path.join(os.homedir(), '.local', 'bin', 'claude.exe'),
    path.join(os.homedir(), '.local', 'bin', 'claude'),
    path.join(os.homedir(), 'AppData', 'Roaming', 'npm', 'claude.cmd'),
    '/usr/local/bin/claude',
    '/opt/homebrew/bin/claude',
  ];
  return candidates.find(p => fs.existsSync(p))
    || (process.platform === 'win32' ? 'claude.cmd' : 'claude');
}

function callClaude(body, overviewOld, oldTitle) {
  const exe = claudeExe();
  const r = spawnSync(exe, ['-p', '--model', MODEL, '--no-session-persistence'], {
    input: PROMPT(body, overviewOld, lengthScale(body.length), oldTitle),
    encoding: 'utf8',
    cwd: STATE_DIR,                              // 中性目录，不加载项目的 CLAUDE.md
    env: { ...process.env, [GUARD]: '1' },
    timeout: 180000,
    windowsHide: true,
    shell: /\.(cmd|bat)$/i.test(exe),            // npm 装的 .cmd 包必须走 shell 才能启动
    maxBuffer: 10 * 1024 * 1024,
  });
  if (r.error || r.status !== 0) {
    log(`claude -p 失败 status=${r.status} err=${r.error} stderr=${(r.stderr || '').slice(0, 500)}`);
    return null;
  }
  const text = (r.stdout || '').trim();
  if (!text) return null;
  const grab = tag => {
    const m = text.match(new RegExp(`【${tag}】([\\s\\S]*?)(?=【(?:标题|做了什么|结论|决策|关键词|概览)】|$)`));
    return m ? m[1].trim() : '';
  };
  const did = grab('做了什么'), concl = grab('结论');
  if (!did && !concl) { log(`模型输出无法解析: ${text.slice(0, 300)}`); return null; }
  return {
    title: grab('标题'),
    did: did || '无',
    concl: concl || '无',
    decision: grab('决策'),
    keywords: grab('关键词'),
    overview: grab('概览'),
  };
}

main();
```

---

## 附录 B：工作原理

```
Claude Code 事件
   │
   ├── PreCompact（上下文即将压缩）──┐
   ├── SessionEnd（退出/登出/clear）─┤
   ├── /summary 手动命令 ────────────┤
   └── SessionStart（打开时）────────┤ 仅清孤儿摘要，不写新摘要
                                     ▼
                          hook 入口脚本（同步，<1s）
                                     │ 读 stdin 拿 session_id / transcript_path / cwd
                                     │ 立即 detached spawn 子进程，父进程退出
                                     ▼
                          后台 worker（异步，20~90s）
                                     │ 加锁 → 读状态 → 增量解析 jsonl
                                     │ 调 claude -p 生成结构化摘要
                                     │ 写入 md / 改名 / 更新索引 / 存状态
                                     ▼
              ~/.claude/hooks/summary-state/<sid>.json   行号 + 文件名 + 标题 + 原始记录路径
              <OUTPUT_DIR>/<日期>_<项目>_<标题>_<8位ID>.md
              <OUTPUT_DIR>/INDEX.md
```

**关键设计**：hook 入口只负责派发，绝不阻塞。压缩和退出都是用户在等的操作，hook 里跑 60 秒的模型调用会让终端卡住。

### 触发时机

| 事件 | 何时 | 段落标记 |
|---|---|---|
| `PreCompact` | 上下文自动/手动压缩前 | `自动压缩前` / `手动压缩前` |
| `SessionEnd` | 退出、登出、`/clear` | `退出时` / `登出时` / `/clear 时` |
| `SessionStart` | 打开 Claude Code | —（只清理孤儿摘要，不写摘要） |
| 手动命令 | 用户执行 `/summary` | `手动总结` |

**刻意不做的**：不在每次任务结束后总结。那样会写成流水账，文件迅速膨胀。

### 增量机制

状态文件记录已读到的 jsonl 行号，每轮只读新增部分，首尾相接不重叠。同一段内容不会被总结两次。

### 标题稳定性

标题存在状态文件里，下一轮连同新增内容一起发给模型，并注明「主题没变就原样输出」。否则模型只看最新一段，会导致每轮都改名。

### 孤儿清理

`DELETE_WITH_TRANSCRIPT` 为真时，`SessionStart` 派发一个独立后台进程（`--sweep-run`）做三件事：读 `~/.claude/projects` 拿全部存活会话 ID → 逐条比对 `summary-state/<完整UUID>.json` → 没有对应原始记录且归属校验通过的就删。

**为什么不用定时任务**：Claude Code 没有常驻进程。装个系统计划任务是另一套要维护、要排障的东西，而用户每次打开 Claude Code 本来就会触发 `SessionStart` —— 顺势扫一遍就够，不需要等待期，也不需要在后台一直挂着。

**为什么 `SessionStart` 放在 `transcript_path` 检查之前**：开场事件不一定带这个字段，先检查就会直接 return，清理永远轮不到执行。

---

## 附录 C：踩坑清单

安装或排查时对照这张表。

| # | 坑 | 症状 | 解法 |
|---|---|---|---|
| 1 | `cleanupPeriodDays` 默认 30 天 | 原始 jsonl 被删，摘要里的链接全废 | 设成 3650，**需重启生效** |
| 2 | Windows 管道带 UTF-8 BOM | **hook 静默失效，日志里一条记录都没有** | `readStdin()` 里 `.replace(/^\uFEFF/, '')`，解析失败必须写日志 |
| 3 | 传了 `--tools ''` | PowerShell 丢弃空参数，`claude -p` 傻等 stdin 直到 ETIMEDOUT | 不要传这个参数 |
| 4 | 标签剥离正则写宽 | 对话里的 HTML / JSX / 代码被整段删掉 | 只匹配已知注入标签，见 `extractText` |
| 5 | 会话没发消息就退出 | 日志刷满 `ENOENT` 堆栈 | 开头 `fs.existsSync` 检查后安静跳过 |
| 6 | 变量先用后定义 | `ReferenceError: Cannot access 'out' before initialization` | 先读旧文件 → 调模型 → 最后算文件名 |
| 7 | 压缩和退出同时触发 | 同一段被总结两次 | `<sid>.lock` 锁文件排队，10 分钟的陈旧锁自动清除 |
| 8 | 内容太少也写 | 堆出一串「本轮无变化」 | `MIN_CHARS = 80` |
| 9 | 涉及文件混入摘要自身 | 列表越滚越乱 | `path.resolve(p).startsWith(path.resolve(SUMMARY_DIR))` 过滤 |
| 10 | 子进程递归触发 hook | 无限套娃 | 环境变量 `CC_SUMMARY_CHILD` 拦住 |
| 11 | 整体覆盖 settings.json | 用户的模型配置、env、权限全丢 | 读 → 备份 → 合并 → 写回 → 复查 |
| 12 | `SessionStart` 里先检查 `transcript_path` | 开场事件没这个字段，清理永远轮不到执行 | 清理逻辑放在这个检查**之前** |
| 13 | 按文件名删摘要 | 把总结目录里手写的笔记一起删了 | 只删 state 登记过、正文含完整 UUID 的 `.md` |
| 14 | 原始记录目录读不到也照删 | 环境异常时清空整个总结目录 | 渐进读取失败就整体放弃，一条 jsonl 都扫不到也放弃 |

---

## 附录 D：跨平台差异

| 项 | Windows | macOS / Linux |
|---|---|---|
| `claude` 路径 | `~/.local/bin/claude.exe` 或 `~/AppData/Roaming/npm/claude.cmd` | `/usr/local/bin/claude`、`/opt/homebrew/bin/claude` |
| `.cmd` 包装 | 需要 `shell: true` | 不适用 |
| 路径分隔符 | JSON 里 `\` 要写成 `\\` | `/` 直接用 |
| 看日志 | `type %USERPROFILE%\.claude\hooks\summary-state\summarize.log` | `cat ~/.claude/hooks/summary-state/summarize.log` |
| 后台窗口 | `windowsHide: true` 避免闪黑框 | 忽略该参数 |

project 目录的转义规则两端一致：把 cwd 里所有非字母数字字符替换成 `-`。
例如 `D:\ai\agent\cc\EverydayConversation` → `D--ai-agent-cc-EverydayConversation`。
