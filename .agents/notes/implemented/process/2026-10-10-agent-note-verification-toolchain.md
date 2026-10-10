# Agent Note: 笔记校验脚本随仓库落地，`npm run verify-notes` 作为提交前门禁

Status: implemented

## Problem

`CLAUDE.md` 要求非平凡改动先写 `.agents/notes/` 笔记、提交前跑 `npm run verify-notes`，但仓库里既没有 `package.json` 也没有校验脚本：门禁只写在文档里，没有可执行物。`.agents/notes/` 下唯一那篇笔记结构是否合规、相对链接是否有效，全靠人眼。

## Decision

把 `write-notes-like-deepseek` skill 的校验脚本（`scripts/*.ts`，11 个）与其 `package.json` 复制进仓库根目录，脚本与命令逐字保留上游形态，不二次加工；skill 目录仍在仓库外，但仓库不再依赖它才能过门禁。

- 门禁即 `npm run verify-notes`：串跑 `verify-agent-note-tree` → `verify-agent-note-format` → `verify-archived` 三线。
- 脚本只依赖 node 内建模块；`agent-note-tree.ts` 从 `cwd` 解析 `.agents/notes`（可用 `AGENT_NOTE_ROOT` 环境变量覆盖）。
- 上游 `package.json` 的入口一律写成 `npx tsx`。本机 node 22 可直接执行 `.ts`（type stripping），故 `node scripts/verify-agent-note-tree.ts` 等价可用，是 `npx` 拉不到包时的退化路径。
- 复制时保持上游脚本清单完整（含 `build-board` / `check-anchors` / `archive-agent-note` / `import-dsh-notes` 等当前没在用的入口），便于后续按整体替换升级。
- 看板产物 `board.html`（~69KB）与 `demo.html`（脱机内联）及 `node_modules/` 一并加进 `.gitignore`，不入库。

## Alternatives considered

- **不落地脚本，直接调用全局 skill 目录里的 `scripts/*.ts`** — 最强理由是零重复、skill 升级自动生效。否掉：skill 目录在仓库外（`~/.claude/skills/...`），换机器或换 skill 版本即失效，而 `npm run verify-notes` 必须能在仓库根目录独立跑通。
- **把 `tsx` 装成 devDependency，脚本改用本地 `tsx`** — 最强理由是 `npx` 首跑要联网拉包（本次冷缓存时就报过 `ENOENT`），本地依赖可离线复现。否掉：会在一个纯配置仓库里引入 `node_modules` 与 lockfile；node 22 已能直接跑 `.ts`，离线路径由 `node scripts/*.ts` 覆盖，代价更小。
- **只复制当前用得到的两个脚本** — 最强理由是减少无关文件。否掉：与上游清单分叉后，每次 skill 升级都要重新判断哪些"用得到"，这个判断成本高于多存几个静态文件。

## Consequences

- **收益**：`CLAUDE.md` 的门禁从"文档里的约定"变成可执行命令；笔记树结构、头块格式、`## Alternatives considered` 的存在由脚本判定，不靠人眼。
- **代价与已知上限**：`scripts/` 是上游 skill 的快照副本，skill 升级需手动重新复制（复制物与上游之间没有自动同步）；`npm run verify-notes` 在 npx 缓存缺失且无网络时会失败，此时只能退化到 `node scripts/*.ts`。

## Verification

- `npm run verify-notes` 输出三行 `ok`、退出码 0（当前 1 篇笔记）。
- `node scripts/verify-agent-note-tree.ts` 同样退出码 0，证明门禁不依赖 `tsx`。
