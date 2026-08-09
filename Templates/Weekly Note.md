---
date: <% moment(tp.file.title, "GGGG-[W]WW").format("YYYY-MM-DD") %>
week: <% moment(tp.file.title, "GGGG-[W]WW").format("GGGG[W]WW") %>
tags:
  - WeeklyNotes
---

<%*
const weekMoment = moment(tp.file.title, "GGGG-[W]WW");
const weekNum = weekMoment.format("WW");
const weekStart = weekMoment.clone().startOf('isoWeek');
const weekDays = [];
for (let i = 0; i < 7; i++) {
    weekDays.push(weekStart.clone().add(i, 'days').format('YYYY-MM-DD'));
}
const weekDayNames = ['周一', '周二', '周三', '周四', '周五', '周六', '周日'];
const prevWeek = weekMoment.clone().subtract(1, 'week').format('GGGG-[W]WW');
const nextWeek = weekMoment.clone().add(1, 'week').format('GGGG-[W]WW');

// 导航栏
tR += `| << [[${prevWeek}|上一周]] | 第 ${weekNum} 周 (${weekDays[0]} ~ ${weekDays[6]}) | [[${nextWeek}|下一周]] >> |\n\n`;
tR += `---\n\n`;
%>

## 本周日记一览

<%*
let diaryTable = "| " + weekDayNames.join(" | ") + " |\n";
diaryTable += "|" + weekDayNames.map(() => ":--:").join("|") + "|\n";
diaryTable += "|" + weekDays.map(d => `[[Calendar/Daily/${d}]]`).join("|") + "|\n\n";
tR += diaryTable;
%>

---

## 本周总结

### 本周回顾

### 待改进


### 下周计划


---

## 本周文件动态

![[Calendar/文件操作.base#最近编辑]]