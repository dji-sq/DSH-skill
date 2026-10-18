---
name: github-extract
description: Extracts a GitHub URL (repo, issue, pull request, commit, file, release) into complete local artifacts AND explains in plain language what the project is trying to do, before presenting any details. Use when given a github.com link to "extract", "pull", "save", "read in full", or "explain", especially when the user says the result was unclear or when the URL carries a #number fragment.
---
# GitHub Extract
## 这个技能要解决的问题
用户丢来一个 GitHub 链接，通常**不是**想要一堆文件内容，而是想弄明白三件事：
1. 这个项目到底是干嘛的？
2. 这次改动想解决什么问题？
3. 为什么用这种做法？
文件、diff、元数据只是**证据**。把证据当结论交出去，等于没提取——读者得到了一堆表，却说不出这个仓库存在的理由。
**完成标准（一句话）**：一个从没听过这个仓库的人，读完前 300 字，能用自己的话复述「它解决什么问题、这次改了什么、为什么这么改」。做不到，就还没提取完。
## 第 0 步：先读「项目」，再读「页面」
在抓 PR / issue 之前，先搞清楚载体本身。跳过这步，后面写得再全也是悬空的。
| 要回答的问题 | 从哪里读 |
| --- | --- |
| 这是什么项目 | repo 的 `description`、`topics`、README 开头 |
| 它补上了什么缺口 / 替代了什么 | README 里每个包的 "Why" 列 |
| 谁在什么场景下用 | README 的 Install / Usage / Develop |
| 这次改动处在什么位置 | PR 的 base 分支、被改的包在清单里的角色 |
**monorepo / 插件集合尤其不能跳**：必须先列出「包名 → 一句话职责」的对照，否则读者无法定位这次改动。若 README 用了大量行话（如 "lifecycle hooks"、"ctx.web seam"），先用一句人话翻译，再进入正文。
## 第 1 步：识别 URL 类型
| URL 形状 | 类型 | 主数据源 |
| --- | --- | --- |
| `github.com/{o}/{r}` | repo | `/readme` + `git/trees?recursive=1` |
| `…/issues/{n}` | issue | `issues/{n}` |
| `…/pull/{n}` | PR | `pulls/{n}` + `pulls/{n}/files` + `pull/{n}.diff` |
| `…/commit/{sha}` | commit | `commits/{sha}` |
| `…/blob/{ref}/{path}` | file | `contents/{path}?ref={ref}`（base64） |
`#1` 是 fragment，不是路径。issue 与 PR 共用一个编号空间：`/issues/1` 若 302 到 `/pull/1`，那它就是 **PR**，必须明说。
## 第 2 步：从 API 取全文，不要跟 HTML 较劲
- 页面 HTML 只给导航栏和截断的正文，正文一律走 `api.github.com`。
- 补丁走 `patch-diff.githubusercontent.com/raw/{o}/{r}/pull/{n}.diff`（`github.com/.../pull/n.diff` 会跨域重定向，必须直连最终地址）。
- `raw.githubusercontent.com` 常不可用，改用 `contents` 的 base64 本地解码。
- 本机 `git` / `curl` 报 TLS `SEC_E_NO_CREDENTIALS` 时**不要去修 shell**，换 harness 的 `web_fetch`；不要把「clone 失败」当成任务受阻。
## 第 3 步：交叉校验
`additions`/`deletions`/`changed_files` ↔ `files` 列表 ↔ `.diff` 的 hunk；PR 的 head SHA ↔ 补丁 ref；`merged` 不能只看 `state=closed`。不一致就继续查（多半是分页或旧补丁），不要写一份自相矛盾的产物。
## 第 4 步：组织成「能读懂」的产物
顺序是硬性的：**解释在前，证据在后**。读者不该翻过 200 行表格才明白这是什么。
1. **它是什么（150–300 字，人话）** — 项目存在的理由、补的是什么缺口。专有名词第一次出现必须一句话解释：`hookkit = 把「写插件才能挂生命周期钩子」变成「写一段 YAML」`。
2. **这次改动想解决什么** — 用「原来会怎样 → 现在怎样」的对比写，不要用文件清单代替。
3. **为什么是这个做法** — 从 PR 描述 / commit message 里提炼取舍与因果链（为什么不选 push 触发、为什么把 registry 当状态）。
4. **改动清单** — 表格：文件 / 状态 / 增删 / 一句话作用。
5. **原文** — PR 描述、commit message 逐字引用，不改写。
6. **完整内容** — 每个文件全文，不写 "……其余省略"。
7. **可迁移的做法** — 只留读者下次真能用上的。
8. **验证与覆盖** — 放最后当附录：`git apply --check` 结果、blob SHA 比对、哪些没抓到。
**写作规则**
- 结论先行：每段第一句就是结论，后面才是依据。
- 表格不承担解释责任。凡是「这是什么」「为什么」，必须用完整句子写。
- 不许用文件名代替功能描述："改了 publish.yml" ❌；"新增手动发布工作流，点按钮才发布，默认只演练" ✅。
- 数字要能追溯来源，但别把校验日志放在开头。
- 用户用什么语言就用什么语言写。
- 反模式：大表格堆砌、把逐文件 diff 当正文、开篇就是元数据、术语不解释。
## 第 5 步：自检
- 只看前 300 字（不含标题表），能否答出「干什么 / 改了什么 / 为什么这样改」？不能就重写。
- 有没有没解释过的术语？
- 是不是把「文件 A、B、C 有变化」当成了改动说明？
- 补丁是否 `git apply --check` 通过？没跑就明确写「未验证」。
## 失败模式
| 症状 | 原因 | 处理 |
| --- | --- | --- |
| 正文写到一半断掉 | 抓了 HTML | 改用 API |
| `404` on raw.githubusercontent | 网络白名单 | 解码 `contents` base64 |
| `403` rate limit | 未认证循环 | 批量、放慢、如实报告缺口 |
| `cross-origin redirect not followed` | `.diff` 跳转 | 直连 `patch-diff…` |
| 文件数与 `changed_files` 不符 | 分页 | `?per_page=100` 重取 |
| 读者反馈「没看懂」 | 只给了证据，没给解释 | 回到第 4 步第 1–3 节重写 |
## 边界
