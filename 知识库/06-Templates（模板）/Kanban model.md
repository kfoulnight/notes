---
kanban-plugin: board
---

<%*
const DATE_FORMAT = "YYYY-MM-DD";
const DAILY_FOLDER = "知识库/01-Daily（日记）";
const targetDate = tp.file.title;

function getPreviousDate(dateText) {
  const match = dateText.match(/^(\d{4})-(\d{2})-(\d{2})$/);
  if (!match) return "";

  const [year, month, day] = match.slice(1).map(Number);
  const date = new Date(Date.UTC(year, month - 1, day));
  if (
    date.getUTCFullYear() !== year ||
    date.getUTCMonth() !== month - 1 ||
    date.getUTCDate() !== day
  ) {
    return "";
  }

  date.setUTCDate(date.getUTCDate() - 1);
  return date.toISOString().slice(0, DATE_FORMAT.length);
}

function extractKanbanSection(content, heading) {
  const lines = content.replace(/\r\n/g, "\n").split("\n");
  const start = lines.findIndex((line) => line.trim() === `## ${heading}`);
  if (start === -1) return "";

  const collected = [];
  for (const line of lines.slice(start + 1)) {
    const trimmed = line.trim();
    if (trimmed.startsWith("## ") || trimmed === "%% kanban:settings") break;
    collected.push(line);
  }

  return collected.join("\n").trim();
}

const previousDate = getPreviousDate(targetDate);
const previousFile = previousDate
  ? app.vault.getAbstractFileByPath(`${DAILY_FOLDER}/${previousDate}.md`)
  : null;
let unfinished = "";
let inProgress = "";
let completed = "";

if (previousFile?.extension === "md") {
  try {
    const content = await app.vault.read(previousFile);
    unfinished = extractKanbanSection(content, "未完成");
    inProgress = extractKanbanSection(content, "进行中");
    completed = extractKanbanSection(content, "已完成");
  } catch (error) {
    console.error("无法读取前一天的日记：", error);
  }
}

tR += `## 未完成

${unfinished}

## 进行中

${inProgress}

## 已完成

${completed}

%% kanban:settings
\`\`\`
{"kanban-plugin":"board","list-collapse":[false,false,false]}
\`\`\`
%%`;
%>