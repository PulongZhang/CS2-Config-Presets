# Agent Note: 个人配置以叠加方式维护，不再保留 cs-backup 快照

Status: implemented

## Problem

仓库根目录承载的是上游模板（当前 v2.6e）。个人玩法偏好此前以整份 `cs-backup/` 快照的形式并存，带来三个问题：

1. 同一份 `auto.cfg` 出现两个副本（模板版 + 个人版），每次上游更新都要人工 diff 才能判断哪条指令是上游新增、哪条是自己改过。
2. 快照停留在 v2.6（2024/8/23），缺 24 条上游新增指令、还含 1 条已被上游删除的废弃指令（`snd_mvp_volume`），实际已经腐烂，不能再当"个人配置"用。
3. 快照是 untracked 目录，没有历史可比，无法回答"我这值是什么时候改的"。

不做区分的话只有两条坏路：永远停在旧版；或合并个人值时把上游更新整片冲掉。

## Decision

个人配置以「上游模板 + 少量个人值」的叠加方式维护：个人值直接写在根目录的 `auto.cfg` / `crosshair.cfg` 内，仓库不再保留 `cs-backup/`。

当前个人值就是全部漂移点，且仅此这些：

- `auto.cfg`：`sensitivity` 1.1545，以及开启模板里默认注释掉的三段绑定——`bind c +duck`（C+空格大跳）、`bind q +jumpthrow`（Q 键左键跳投）与 `bind h +jumpthrow2`（H 键右键跳投），后两段各连同其 6 行 `alias`（跳投合计 14 行，另加各自小节标题行的"未开启→已开启"字样）。这三段是模板自带的可选项，个人选择开启，故计入漂移。注意 `bind h` 位于模板 `bind h "switchhands"`（切换左右手持枪）之后，同键后者被覆盖——这是模板里该段默认注释掉的原因，个人接受。
- `auto.cfg`：`fps_max` 200（模板 400）。取 200 的理由：显示器的 `-refresh 200` 就写在 `启动项.txt`（即当前刷新率上限），再往上只产出撕裂与发热，不产出可见流畅度；200 同时对齐 `hlae.cfg` 的录制限制帧数，主配置与录制配置不再各说各话。`fps_max_ui` 120 与 `fps_max_tools` 120 保持模板值，主界面与工具跑满 200 帧无收益、白耗电。
- `auto.cfg`：`cl_hud_telemetry_frametime_show` 与 `cl_hud_telemetry_ping_show` 由模板的 1（仅条件恶劣时显示）改为 2（总是显示），对应"常显 FPS 与延迟"。只改"是否显示"这两条，告警阈值（`cl_hud_telemetry_frametime_poor` 6.94、`cl_hud_telemetry_ping_poor` 100）保持模板值。
- `crosshair.cfg`：`cl_crosshairsize` 0、`cl_crosshairgap` -5、`cl_crosshairthickness` 1.0、`cl_crosshairdot` 1、`cl_crosshairalpha` 255、`cl_crosshair_drawoutline` 1
- `cs2_video.txt`：整份等于 `Steam/userdata/<id>/730/local/cfg/cs2_video.txt` 的逐字节副本（见下）

其余一律等于上游模板值——包括 `rate` 524288、`cl_crosshair_friendly_warning` 1、`cl_hud_telemetry_frametime_poor` 6.94、`mm_dedicated_search_maxping` 120、`m_yaw` 0.022，以及全部 `snd_*` 音量。这些曾一度被个人化成更宽松的值，已按"废弃默认值/非手调项回归模板"处理。

#### cs2_video.txt：镜面副本，不再是参考留存

`cs2_video.txt` 从快照提到仓库根目录后，定位由"视频配置参考留存"改为**当前生效视频设置的镜面副本**（2026-10-12）。理由：它在仓库里的唯一消费方就是"我现在的画面设置是什么"，而参考留存这个定位让它在被同步进快照后 30 项设置里漂移了 7 项（`cpu_level` 2↔3、`fullscreen_min_on_focus_loss` 1↔0、`r_low_latency` 2↔1、`refresh` 170/1000↔0/0、`Version` 15↔16、`DeviceID` 8089↔10400），`DeviceID` 甚至指向别台机器的显卡——它已经不是任何人的配置。

约定：同步方式为从 `.../730/local/cfg/cs2_video.txt` 整文件 `cp` 覆盖（游戏会按 key 字典序重写该文件，故不做逐行编辑，避免行序差异带来的伪 diff）。它仍然不进 `auto.cfg`、不由任何 cfg `exec`，但从此必须与游戏内当前设置保持一致——游戏内改过视频设置就重新覆盖一次并提交。

上游模板无此文件，故它不受"上游同步重放清单"管辖，同步时直接保留本地副本。

个人工作分支为 `origin/personal`；上游仓库同步到 `origin/master`。

### 上游同步流程

拉取上游 `master` → 用上游模板覆盖根目录文件 → 重放上面列出的 23 + 6 行个人改动 → 提交到 `personal`。这份清单即重放清单，两者必须一起更新。

## Alternatives considered

- **保留 `cs-backup/` 快照** — 最强的理由是个人值与模板物理隔离，误改模板的风险为零，且个人值集中在一处好查。否掉：同一参数两份声明、上游每加一条指令都要人工对账；且该快照已停在 v2.6（缺新增、含废弃指令），隔离换来的确定性早被腐烂吃掉。
- **`cs2_video.txt` 保持"参考留存"** — 最强的理由是不必为每次游戏内改动视频设置而提交一次仓库，且它能记录一份"曾经调好的"画面设置。否掉：它已被同步进快照并漂移 7 项（含指向别台机器的 `DeviceID`），留下的是既非当前生效也非有意保留的中间态；而它唯一的实际用途本就是"我现在是什么设置"，镜面副本才自洽。
- **个人值放独立 `.cfg`，由 `auto.cfg` 末尾 `exec`** — 最强理由是上游整份覆盖 `auto.cfg` 时个人值不受影响，是最彻底的解耦。否掉：CS2 的 `exec` 生效顺序会随上游重排而错位，准星与灵敏度必须在模板对应行之后才生效，调试成本高于收益；本仓库是个人 fork，直接改模板更直接。
- **全程使用模板默认值** — 最强理由是零维护、上游同步零成本。否掉：`sensitivity` 与准星参数是实打实的手感设置，不能丢。`fps_max` 曾按此思路长期停在模板 400，2026-10-13 判为代价不划算：与本机 `-refresh 200` 不符，且让 `auto.cfg` 与 `hlae.cfg` 各说各话。
- **`fps_max` 改为 400 以上（如 0 不限或 500）** — 最强理由是 CS2 帧数越高输入延迟越低，职业选手普遍不设上限。否掉：本机刷新率上限就是 200，超出部分不进画面只进延迟统计与发热，收益不可见。

## Consequences

- **收益**：单一事实来源——根目录文件就是最终生效配置；上游新增指令自动继承；个人漂移点一眼可数（当前 auto.cfg 23 行、crosshair.cfg 6 行）。
- **漂移清单的口径**：只收录有理由的个人值，不收录并入快照时的残留。`fps_max` 曾按此口径判为残留（个人值 300 随 v2.6 快照并入，并非手调手感值，而上游模板自 bb325c3 起为 400），故 2026-10-13 前一直回归模板 400；2026-10-13 重新个人化为 200，理由不再是"手感"而是上述刷新率对齐，故自此计入清单。
- **代价与已知上限**：上游若改动同一行（例如上游调整 `sensitivity` 默认值、或重排 `crosshair.cfg`），合并即冲突，必须人工重放而非自动覆盖。信号：上游同步后 `git diff` 出现预期外的行，说明漂移清单已过期，本笔记必须同批修正。

## Verification

- `git diff origin/master -- auto.cfg crosshair.cfg` 只输出 `sensitivity`、`fps_max`、大跳/跳投三段（18 行）、两条 telemetry 显示开关（2 行）与 6 项准星参数，共 56 行改动（每处 1 删 1 增，`git diff --stat` 报 28 insertions / 28 deletions）。
- `grep -n '^crosshair ' auto.cfg` 输出 `crosshair 1`（启用准星，个人准星由 `crosshair.cfg` 提供）。
- `git branch -vv` 显示 `personal` 跟踪 `origin/personal`。
- `diff cs2_video.txt "Steam/userdata/891305550/730/local/cfg/cs2_video.txt"` 无输出（镜面副本成立）。
