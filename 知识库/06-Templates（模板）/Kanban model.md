---
kanban-plugin: board
---

<%*
const DATE_FORMAT = "YYYY-MM-DD";
const yesterdayDate = tp.date.now(DATE_FORMAT, -1);

// 日记所在的库内路径；留空时会在整个库中查找昨天的日记。
const DAILY_FOLDER = "";

function escapeRegExp(text) {
  return text.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

function extractKanbanSection(content, heading) {
  const pattern = new RegExp(
    `^##\\s+${escapeRegExp(heading)}\\s*\\n([\\s\\S]*?)(?=^##\\s+|^%%\\s*kanban:settings|$)`,
    "m"
  );
  const match = content.replace(/\r\n/g, "\n").match(pattern);
  return match ? match[1].trim() : "";
}

async function findYesterdayDailyNote() {
  if (DAILY_FOLDER) {
    const file = app.vault.getAbstractFileByPath(
      `${DAILY_FOLDER}/${yesterdayDate}.md`
    );
    if (file) return file;
  }

  return app.vault.getMarkdownFiles().find(
    (file) => file.basename === yesterdayDate
  );
}

const yesterdayFile = await findYesterdayDailyNote();
let unfinished = "";
let inProgress = "";

if (yesterdayFile) {
  const content = await app.vault.read(yesterdayFile);
  unfinished = extractKanbanSection(content, "未完成");
  inProgress = extractKanbanSection(content, "进行中");
}

tR += `## 未完成

${unfinished}

## 进行中

${inProgress}

## 已完成



%% kanban:settings
\`\`\`
{"kanban-plugin":"board","list-collapse":[false,false,false]}
\`\`\`
%%`;
%>