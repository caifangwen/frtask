这是一个关于 Obsidian Dataview 配置的深度问题，不需要创建文件，我直接给出详细的配置方案。

---

## Dataview 细化配置

### 前置：支线文件的 Frontmatter 规范

Dataview 依赖 YAML frontmatter 读取结构化数据。先统一支线文件头部格式：

```yaml
---
类型: 支线
状态: 进行中          # 进行中 | 等待 | 待启动 | 已完成
优先级: 高            # 高 | 中 | 低
开始日期: 2026-03-01
预计完成: 2026-03-15
负责人: 自己
标签: [开发, 客户端]
---
```

> **关键**：Dataview 读取的是 frontmatter 字段，不是正文内容。字段名要保持完全一致（区分全角/半角）。

---

### `_INDEX.md` 完整 Dataview 配置

#### 1. 进行中支线（带进度条）

````markdown
```dataview
TABLE WITHOUT ID
  link(file.link, file.aliases[0] + " · " + 文件名) AS "支线",
  状态 AS "状态",
  优先级 AS "优先",
  开始日期 AS "开始",
  预计完成 AS "截止",
  标签 AS "标签"
FROM "支线"
WHERE 状态 = "进行中"
SORT 优先级 ASC
```
````

> `优先级` 排序：高→中→低，需要改成数字字段才能真正排序（见进阶方案）。

#### 2. 等待/阻塞看板

````markdown
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  阻塞原因 AS "阻塞原因",
  预计完成 AS "等待截止"
FROM "支线"
WHERE 状态 = "等待"
SORT 预计完成 ASC
```
````

对应支线 frontmatter 加一个字段：`阻塞原因: 等待甲方回复`

#### 3. 待启动队列

````markdown
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  优先级 AS "优先级",
  标签 AS "类型"
FROM "支线"
WHERE 状态 = "待启动"
SORT 优先级 ASC
```
````

#### 4. 最近更新动态（按文件修改时间）

````markdown
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  状态 AS "状态",
  file.mtime AS "最后更新"
FROM "支线"
WHERE 状态 != "已完成"
SORT file.mtime DESC
LIMIT 5
```
````

#### 5. 即将到期预警（7天内）

````markdown
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  状态 AS "状态",
  预计完成 AS "截止日期",
  (date(预计完成) - date(today)).days + "天" AS "剩余"
FROM "支线"
WHERE 状态 != "已完成"
  AND date(预计完成) <= date(today) + dur(7 days)
  AND date(预计完成) >= date(today)
SORT 预计完成 ASC
```
````

#### 6. 已完成归档（最近10条）

````markdown
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  开始日期 AS "开始",
  file.mtime AS "完成时间",
  标签 AS "类型"
FROM "支线" OR "归档"
WHERE 状态 = "已完成"
SORT file.mtime DESC
LIMIT 10
```
````

---

### 进阶：优先级真正可排序

将 frontmatter 改为：

```yaml
优先级: 1       # 1=高 2=中 3=低
优先级标签: 高   # 用于显示
```

查询时：

```
SORT 优先级 ASC   # 1排在最前
```

---

### Dataview JS：全局仪表盘（`_INDEX.md` 顶部）

用 `dataviewjs` 渲染一个简单的状态统计：

````markdown
```dataviewjs
const pages = dv.pages('"支线"');
const stats = {
  进行中: pages.where(p => p.状态 === "进行中").length,
  等待: pages.where(p => p.状态 === "等待").length,
  待启动: pages.where(p => p.状态 === "待启动").length,
  已完成: pages.where(p => p.状态 === "已完成").length,
};

dv.paragraph(
  `📊 **支线总览** — ` +
  `🟢 进行中 **${stats.进行中}** ｜ ` +
  `🟡 等待 **${stats.等待}** ｜ ` +
  `⬜ 待启动 **${stats.待启动}** ｜ ` +
  `✅ 已完成 **${stats.已完成}**`
);
```
````

效果：`📊 支线总览 — 🟢 进行中 3 ｜ 🟡 等待 1 ｜ ⬜ 待启动 2 ｜ ✅ 已完成 7`

---

### 支线文件模板（Templater 插件）

在 Templater 设置的模板文件夹里新建 `支线模板.md`：

```markdown
---
类型: 支线
状态: 待启动
优先级: 2
优先级标签: 中
开始日期: <% tp.date.now("YYYY-MM-DD") %>
预计完成: <% tp.date.now("YYYY-MM-DD", 14) %>
负责人: 自己
阻塞原因: 
标签: []
---

# <% tp.file.title %>

## 目标
> 

## 当前进展
- [ ] 步骤一

## 工作记录
### <% tp.date.now("YYYY-MM-DD") %>
- 

## 待解决 / 阻塞

## 关联资源
- [[_INDEX]]
```

新建支线时：`Cmd+P` → `Templater: Create new note from template` → 选模板，自动填充日期。

---

### 常见问题

|问题|原因|解决|
|---|---|---|
|查询结果为空|frontmatter 字段名有空格/全角冒号|检查 `:` 是英文冒号|
|日期排序不对|日期格式不是 `YYYY-MM-DD`|统一日期格式|
|剩余天数报错|Dataview 版本较旧不支持 `.days`|升级到 0.5.x+|
|归档文件不出现|`FROM` 范围没包含归档文件夹|加 `OR "归档"`|