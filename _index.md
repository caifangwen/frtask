# 总任务看板
更新时间：{{date}}

## 全局仪表盘
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

## 进行中支线
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  状态 AS "状态",
  优先级 AS "优先",
  开始日期 AS "开始",
  预计完成 AS "截止",
  标签 AS "标签"
FROM "支线"
WHERE 状态 = "进行中"
SORT 优先级 ASC
```

## 等待/阻塞看板
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  阻塞原因 AS "阻塞原因",
  预计完成 AS "等待截止"
FROM "支线"
WHERE 状态 = "等待"
SORT 预计完成 ASC
```

## 待启动队列
```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "支线",
  优先级 AS "优先级",
  标签 AS "类型"
FROM "支线"
WHERE 状态 = "待启动"
SORT 优先级 ASC
```

## 最近更新动态
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

## 即将到期预警（7天内）
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

## 已完成归档
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

## 今日重点
> 优先推进支线-A，支线-B 阻塞中暂缓