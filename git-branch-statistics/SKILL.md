---
name: git-branch-statistics
description: 统计当前 Git 分支自建分支以来的自有提交代码量与结构：新增行数、删除行数、总变更行数、净增行数，以及新增文件数、删除文件数、修改文件数；附带 AI 代码占比与作者贡献分布。当用户要求查看分支代码统计、代码量、改动行数、文件数、AI 占比、作者贡献时触发。
when_to_use: 用户询问当前分支/本次迭代写了多少代码、涉及多少文件、AI 占比多少、谁贡献了多少。
---

# 分支代码统计

## Overview

统计**当前分支自己写的提交**（分支创建点 → HEAD，排除 merge 带入的他人提交），输出行数、文件数、AI 占比、作者贡献。

## 统计口径（核心，不可随意改动）

**范围**：`<分支创建点>..HEAD`，创建点用 `git merge-base --fork-point $BASE HEAD`（回退 `git merge-base`）。
**不与主干做内容比对**——主干上别人的提交与本分支无关，`git diff $BASE...HEAD` 这类口径一律不采用。

| 维度 | 口径 | 为什么 |
|:---|:---|:---|
| 行数 | 不累加 | 同一行被改 3 次算 1 行 |
| 新增页面数 | 针对 /src/router/ 或者 /pages.json 下的对象计算新增的页面 | 一个页面可能是多个文件组成的， 通过修改的文件来匹配路由文件， 匹配到一条则计算为一个页面，子路由也计算为一个页面，不累加 |
| 修改页面数 | 针对 /src/router/ 或者 /pages.json 下的对象计算修改的已有页面 | 一个页面可能是多个文件组成的， 通过修改的文件来匹配路由文件， 匹配到一条则计算为一个页面，子路由也计算为一个页面，不累加 |
| 总变更行数 | 新增 + 删除 | 不单独列"修改行数"：git 只有新增/删除两类事件，"修改"只是两者同现，单列会重复计数 |
| 净增行数 | 新增 − 删除 | 分支对代码体量的实际影响 |
| 文件归类 | 每个文件只归一类，按分支内**演变结果**判定 | 见下表 |
| 重命名 R | 归入"修改"，按目标路径计 1 个文件 | 源路径不重复计数 |
| 复制 C | 归入"新增" | 确实产生了新文件 |
| 分支内新建后又删除 | 单列为"净变更为零"，不计入三类 | 避免新增数虚高 |
| 先删后建 / 改回原样 | 归入"修改" | 文件确实被动过，即使最终内容与主干一致 |
| 排除项 | `.qoder/repowiki/`、`dist/`、`node_modules/`、`package-lock.json`、`yarn.lock`、`pnpm-lock.yaml`、`go.sum`、`*.min.js`、`*.min.css` | 自动生成内容 |
| merge commit | 排除（`--no-merges`） | 其变更来自被合入的分支，非本分支工作量 |

## 执行步骤

### 1. 跑 git 统计

在仓库内任意目录执行整段脚本（脚本自身会 `cd` 到仓库根）。用户要换基准分支时用 `BASE=<分支>` 前缀；要改 AI 标记规则时用 `AIMARK='<正则>'`。

```bash
cd "$(git rev-parse --show-toplevel)" || { echo "当前目录不是 git 仓库"; exit 1; }
CUR=$(git branch --show-current)
BASE=${BASE:-$(git rev-parse --verify --quiet origin/master >/dev/null && echo origin/master || (for b in master main develop; do git rev-parse --verify --quiet "$b" >/dev/null && { echo "$b"; break; }; done))}
[ -z "$BASE" ] && { echo "未找到基准分支，请用 BASE=<分支> 指定"; exit 1; }
FORK=$(git merge-base --fork-point "$BASE" HEAD 2>/dev/null || git merge-base "$BASE" HEAD)
[ -z "$FORK" ] && { echo "找不到 $CUR 相对 $BASE 的分支创建点"; exit 1; }
RANGE="$FORK..HEAD"
COUNT=$(git rev-list --count $RANGE --no-merges)
[ "$COUNT" = 0 ] && { echo "$CUR 相对 $BASE 无自有提交"; exit 1; }
REPO=$(basename "$(git rev-parse --show-toplevel)")
AIMARK=${AIMARK:-'\[AI\]|Co-authored-by: Qoder'}

echo "分支 $CUR | 创建点 ${FORK:0:7}（相对 $BASE）| 自有提交 $COUNT 次（不含 merge 带入的他人提交）"
echo "REPO: $REPO"

echo "ROWS:"
git log $RANGE --no-merges --numstat --format="" \
  | awk -F'\t' '$1 ~ /^[0-9]+$/ && $3 !~ /\.qoder\/repowiki\/|(^|\/)(dist|node_modules)\// && $3 !~ /(^|\/)(package-lock\.json|yarn\.lock|pnpm-lock\.yaml|go\.sum)$/ && $3 !~ /\.min\.(js|css)$/ {a+=$1; d+=$2} END {printf "  新增 %d 删除 %d 总变更 %d 净增 %d\n", a, d, a+d, a-d}'

echo "FILES:"
git log $RANGE --no-merges --name-status --format="" --reverse \
  | awk -F'\t' '
  NF < 2 { next }
  {
    st = substr($1, 1, 1)
    if (NF >= 3) { p = $3 } else { p = $2 }
    if (p ~ /\.qoder\/repowiki\/|(^|\/)(dist|node_modules)\//) next
    if (p ~ /(^|\/)(package-lock\.json|yarn\.lock|pnpm-lock\.yaml|go\.sum)$/) next
    if (p ~ /\.min\.(js|css)$/) next
    prev = (p in kind) ? kind[p] : ""
    if (st == "A")      kind[p] = (prev == "" ? "A" : (prev == "Z" || prev == "D" ? "M" : prev))
    else if (st == "C") kind[p] = (prev == "" ? "A" : prev)
    else if (st == "R") kind[p] = (prev == "" ? "M" : prev)
    else if (st == "M") kind[p] = (prev == "" ? "M" : prev)
    else if (st == "D") kind[p] = (prev == "A" ? "Z" : "D")
  }
  END {
    for (f in kind) { c[kind[f]]++; n++ }
    printf "  新增文件 %d  删除文件 %d  修改文件 %d  涉及文件合计 %d\n", c["A"], c["D"], c["M"], n
    if (c["Z"]) printf "  （另有 %d 个文件分支内新建后又删除，净变更为零，不计入以上三类）\n", c["Z"]
  }'

echo "AUTHORS:"
git log $RANGE --no-merges --numstat --format="@%an" \
  | awk -F'\t' '
  /^@/ {au=substr($0,2); seen[au]=1; n[au]++; next}
  $1 ~ /^[0-9]+$/ && $3 !~ /\.qoder\/repowiki\/|(^|\/)(dist|node_modules)\// && $3 !~ /(^|\/)(package-lock\.json|yarn\.lock|pnpm-lock\.yaml|go\.sum)$/ && $3 !~ /\.min\.(js|css)$/ {a[au]+=$1; d[au]+=$2}
  END {for (x in seen) printf "  %s\t%d次\t+%d\t-%d\n", x, n[x], a[x], d[x]}' | sort -t$'\t' -k4,4nr

echo "AI-MARKERS:"
for h in $(git log $RANGE --no-merges --format="%H"); do
  m=$(git log -1 --format="%s%n%b" "$h" | grep -ciE "$AIMARK")
  s=$(git show --numstat --format="" "$h" | awk -F'\t' '$1 ~ /^[0-9]+$/ && $3 !~ /\.qoder\/repowiki\/|(^|\/)(dist|node_modules)\// && $3 !~ /(^|\/)(package-lock\.json|yarn\.lock|pnpm-lock\.yaml|go\.sum)$/ && $3 !~ /\.min\.(js|css)$/ {a+=$1; d+=$2} END {print a+0, d+0}')
  [ "${m:-0}" -gt 0 ] && echo "AI $s" || echo "HUMAN $s"
done | awk '{if($1=="AI"){aa+=$2;ac++}else{ha+=$2;hc++}} END {t=aa+ha
  if (ac==0) { printf "  带 AI 标记的提交 0/%d → 无法判定，需 Qoder Metrics API（不要报 0%%）\n", hc; exit }
  printf "  AI提交 %d/%d  AI新增 %d行 人工新增 %d行 AI新增占比 %.1f%%（估算）\n", ac, ac+hc, aa, ha, (t>0?aa*100/t:0)}'

echo "API-INPUT:"
echo "  start_date=$(git log $RANGE --no-merges --format='%cI' | sort | sed -n '1p')"
echo "  end_date=$(git log $RANGE --no-merges --format='%cI' | sort | sed -n '$p')"
echo "  repo_name=$REPO"
```

二进制文件在 `--numstat` 中为 `-	-	path`，`$1 ~ /^[0-9]+$/` 已自动跳过。

### 2. AI 占比：优先用 Qoder Metrics API

上一步的 `AI-MARKERS` 只是估算。有组织凭证时用 API 精确值替换。

前置：`QODER_API_KEY`、`QODER_ORG_ID`（管理员需开通「AI 代码数据分析」）；`QODER_BASE_URL` 默认 `https://api.qoder.com`，国内版 `https://api.qoder.com.cn`。时间取 `API-INPUT` 的 `start_date`/`end_date`，跨度 ≤ 90 天（超了就分段）。

```bash
curl -sS -f -H "Authorization: Bearer $QODER_API_KEY" \
  "$QODER_BASE_URL/v1/organizations/$QODER_ORG_ID/ai-code/stats/overview?start_date=$START&end_date=$END&repo_name=$REPO"
```

取 `aiShareRate`（= `aiLinesChanged / totalLinesChanged`）、`aiLinesChanged`、`totalLinesChanged`、`agentEditCount`、`tabCompletionCount`、`messageCount`。

**AI 占比三态规则（重要）**：
- API 成功 → 报 API 值，标注"Qoder 组织后端统计，T+1 延迟"。
- 无 API 但有 AI 标记 → 报估算值，必须标注"估算，仅基于提交标记"。
- 无 API 且标记数为 0 → **报"无法统计"**，说明原因（未配置凭证 + 提交无 AI 标记），**绝不允许输出 `0%`**——那会被读成"团队没用 AI"。

严禁把 API Key 写进仓库、脚本或报告。

## 输出格式

```markdown

### 口径说明
- 范围：分支创建点 → HEAD 的自有提交，已排除 merge 带入的主干他人提交
- 行数：逐提交累加（同一行多次修改重复计数），≥ 与主干 diff 的口径
- 文件：每个文件按分支内演变结果归一类；重命名算修改，先删后建算修改
- 已排除：.qoder/repowiki、lock 文件、dist/、node_modules/、*.min.*
- 时间范围 <start> → <end>，仓库 <repo_name>

## 分支代码统计：<CUR>（自建分支以来，创建点 <FORK 短哈希>）

| 指标 | 数值 |
|:---|---:|
| 自有提交数 | N 次 |
| 新增行数 | +X |
| 删除行数 | -X |
| 总变更行数 | X |
| 净增行数 | +X |
| 新增文件数 | X |
| 删除文件数 | X |
| 修改文件数 | X（含重命名 X 个） |
| 新增页面数 | X |
| 修改页面数 | X |

### AI 代码占比
数据源：Qoder AI Code Metrics API / 提交标记估算 / 无法统计
`aiShareRate` = X%　AI 变更行 X / 总变更行 X

### 作者贡献
| 作者 | 提交数 | 新增行数 | 删除行数 |
|:---|---:|---:|---:|

```

## 失败处理

| 情况 | 处理 |
|:---|:---|
| 不在 git 仓库 | 提示"当前目录不是 git 仓库，无法统计" |
| 无 origin/master 且无 master/main/develop | 提示指定 `BASE=<分支>`，不要猜 |
| `--fork-point` 失败（reflog 缺失） | 脚本已自动回退 `git merge-base`，无需干预 |
| 当前即在基准分支 | 提示"无自有提交"，改问用户想统计哪段范围 |
| 创建点异常古老（提交数远超预期） | 说明可能本地主干过旧，建议 `BASE=origin/<分支>` 或让用户指定起点 |
| API 401/403 | AI 占比降级为标记估算或"无法统计"，说明"Key 无效或组织未开通该能力" |
| API 网络错误 | git 统计照常输出，AI 部分标注"API 不可用" |
| 其他 git 报错 | 原样给出报错与可能原因，不做静默兜底 |

## 注意事项

- 不要改用 `git diff $BASE...HEAD` 统计文件或行数——那是与主干比对，会把主干他人改动算进来、并漏掉本分支内"建了又删"的文件。
- 不要用 `feat:`/`fix:` 等 conventional commit 前缀猜 AI 归属，只认 `AIMARK` 里的标记。
- 公共迭代分支上作者贡献会有多人，报告需注明"包含 N 位开发者的提交，非个人数据"。
- 用户可指定起点提交、时间范围或只统计某些路径，覆盖默认口径，但须在报告中标明实际口径。
- Windows 需在 Git Bash 执行（依赖 `awk`/`sed`/`sort`）；PowerShell 缺这些工具时提示改用 Git Bash。