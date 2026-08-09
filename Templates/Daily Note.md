---
date: <% moment(tp.file.title, "YYYY-MM-DD").format("YYYY-MM-DD") %>
week: <% moment(tp.file.title, "YYYY-MM-DD").format("GGGG[W]WW") %>
tags:
  - DailyNotes
---

<%*
const fileDate = moment(tp.file.title, "YYYY-MM-DD");
const prevDay = fileDate.clone().subtract(1, 'days').format('YYYY-MM-DD');
const nextDay = fileDate.clone().add(1, 'days').format('YYYY-MM-DD');
const prevWeek = fileDate.clone().subtract(1, 'week').format('GGGG-[W]WW');
const nextWeek = fileDate.clone().add(1, 'week').format('GGGG-[W]WW');
const todayStr = fileDate.format('YYYY年M月D日, dddd');
const currentWeek = fileDate.clone().format('GGGG-[W]WW');

tR += `| [[Calendar/Weekly/${prevWeek}|上周]] | [[Calendar/Daily/${prevDay}|上天]] | ${todayStr} | [[Calendar/Daily/${nextDay}|下天]] | [[Calendar/Weekly/${currentWeek}|本周]] | [[Calendar/Weekly/${nextWeek}|下周]] |\n\n`;
tR += `---\n\n`;
%>

### 日记

---

### Summary

---

### 今日文件状态

> ![[Calendar/文件操作.base#今日创建]]
