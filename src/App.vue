<script setup lang="ts">
import { computed, reactive, ref, watch } from "vue";

type Field = {
  key: string;
  label: string;
  type?: "number" | "date" | "select";
  options?: readonly string[];
};

type RecordItem = {
  id: string;
  status: string;
  notes: string;
  createdAt: string;
  [key: string]: string | number;
};

type BatchSnapshot = {
  id: string;
  name: string;
  beforeStatus: string;
};

type BatchDraft = {
  id: string;
  targetStatus: string;
  reason: string;
  stagedAt: string;
  snapshots: BatchSnapshot[];
};

type BatchHistoryEntry = BatchDraft & {
  publishedAt: string;
  undone: boolean;
};

type MessageType = "error" | "success" | "info";

const project = {
  "number": 21,
  "folder": "hxwl/frontend/hxwlfront-21",
  "framework": "vue",
  "title": "油站网点地图管理",
  "subtitle": "维护油站位置、营业状态和库存摘要。",
  "industry": "石油",
  "stack": [
    "Vue3",
    "Vite",
    "TypeScript",
    "Element Plus",
    "Leaflet"
  ],
  "storageKey": "hxwlfront-21-station-map",
  "formTitle": "新增油站",
  "primaryAction": "保存油站",
  "entityLabel": "油站",
  "statuses": [
    "营业中",
    "暂停营业",
    "库存紧张"
  ],
  "filters": [
    "全部区域",
    "东区",
    "西区",
    "机场线"
  ],
  "fields": [
    {
      "key": "station",
      "label": "油站名称"
    },
    {
      "key": "area",
      "label": "区域",
      "type": "select",
      "options": [
        "东区",
        "西区",
        "机场线"
      ]
    },
    {
      "key": "stock",
      "label": "库存摘要L",
      "type": "number"
    },
    {
      "key": "manager",
      "label": "负责人"
    }
  ],
  "records": [
    {
      "station": "东区一站",
      "area": "东区",
      "stock": 36000,
      "manager": "刘站长",
      "status": "营业中",
      "notes": "库存正常"
    },
    {
      "station": "机场快线站",
      "area": "机场线",
      "stock": 9000,
      "manager": "王站长",
      "status": "库存紧张",
      "notes": "柴油待补"
    }
  ],
  "metricLabels": [
    "油站数",
    "营业中",
    "库存紧张"
  ]
} as const;

const fields = project.fields as readonly Field[];
const statuses = [...project.statuses];

function createBlank() {
  return Object.fromEntries(fields.map((field) => [field.key, field.type === "number" ? 0 : ""]));
}

function loadRecords(): RecordItem[] {
  const raw = localStorage.getItem(project.storageKey);
  if (!raw) {
    return project.records.map((record, index) => ({
      ...record,
      id: `seed-${index + 1}`,
      createdAt: new Date(Date.now() - index * 86400000).toISOString()
    })) as RecordItem[];
  }
  try {
    return JSON.parse(raw) as RecordItem[];
  } catch {
    return [];
  }
}

const draftStorageKey = `${project.storageKey}-batch-draft`;
const historyStorageKey = `${project.storageKey}-batch-history`;
const filterStorageKey = `${project.storageKey}-filter`;

function loadJSON<T>(key: string, fallback: T): T {
  const raw = localStorage.getItem(key);
  if (!raw) return fallback;
  try {
    return JSON.parse(raw) as T;
  } catch {
    return fallback;
  }
}

const records = ref<RecordItem[]>(loadRecords());
const form = reactive<Record<string, string | number>>(createBlank());
const note = ref("");
const filter = ref(
  loadJSON<string>(filterStorageKey, project.filters[0] as string)
);

const selectedIds = ref<string[]>([]);
const batchTarget = ref(statuses[0]);
const batchReason = ref("");
const batchDraft = ref<BatchDraft | null>(loadJSON<BatchDraft | null>(draftStorageKey, null));
const batchHistory = ref<BatchHistoryEntry[]>(loadJSON<BatchHistoryEntry[]>(historyStorageKey, []));
const message = ref<{ type: MessageType; text: string } | null>(null);
let messageTimer: ReturnType<typeof setTimeout> | undefined;

function notify(type: MessageType, text: string) {
  message.value = { type, text };
  clearTimeout(messageTimer);
  messageTimer = setTimeout(() => {
    message.value = null;
  }, 4200);
}

watch(filter, (value) => localStorage.setItem(filterStorageKey, JSON.stringify(value)));
watch(batchDraft, (value) => localStorage.setItem(draftStorageKey, JSON.stringify(value)), { deep: true });
watch(batchHistory, (value) => localStorage.setItem(historyStorageKey, JSON.stringify(value)), { deep: true });

const filteredRecords = computed(() => {
  if (filter.value.startsWith("全部")) return records.value;
  return records.value.filter((record) => Object.values(record).includes(filter.value));
});

const metrics = computed(() => {
  const total = records.value.length;
  const second = records.value.filter((record) => record.status === statuses[1]).length;
  const third = records.value.filter((record) => record.status === statuses[2]).length;
  const numberValues = records.value.flatMap((record) =>
    fields.filter((field) => field.type === "number").map((field) => Number(record[field.key] || 0))
  );
  const sum = numberValues.reduce((acc, value) => acc + value, 0);
  return [total, second || sum, third || Math.round(sum / Math.max(total, 1))];
});

const chartRows = computed(() => statuses.map((status) => ({
  status,
  value: records.value.filter((record) => record.status === status).length
})));

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

function persist() {
  localStorage.setItem(project.storageKey, JSON.stringify(records.value));
}

function nextStatus(status: string) {
  const index = statuses.indexOf(status);
  return statuses[(index + 1) % statuses.length];
}

function primaryText(record: RecordItem) {
  const first = fields[0];
  const second = fields[1];
  return [record[first.key], record[second.key]].filter(Boolean).join(" / ") || project.entityLabel;
}

function submit() {
  records.value = [
    {
      ...form,
      id: crypto.randomUUID(),
      status: statuses[0],
      notes: note.value || "暂无备注",
      createdAt: new Date().toISOString()
    } as RecordItem,
    ...records.value
  ];
  Object.assign(form, createBlank());
  note.value = "";
  persist();
}

function flow(record: RecordItem) {
  record.status = nextStatus(record.status);
  persist();
}

function remove(id: string) {
  records.value = records.value.filter((record) => record.id !== id);
  selectedIds.value = selectedIds.value.filter((selectedId) => selectedId !== id);
  persist();
}

function stationName(id: string) {
  const target = records.value.find((record) => record.id === id);
  return target ? primaryText(target) : "已删除站点";
}

function draftCurrentStatus(id: string) {
  return records.value.find((record) => record.id === id)?.status ?? "";
}

function selectFiltered() {
  selectedIds.value = filteredRecords.value.map((record) => record.id);
}

function clearSelection() {
  selectedIds.value = [];
}

function stageBatch() {
  if (batchDraft.value) {
    notify("info", "已有暂存批次，请先提交或撤回暂存。");
    return;
  }
  const reason = batchReason.value.trim();
  if (selectedIds.value.length === 0) {
    notify("error", "请先勾选需要批量发布的站点。");
    return;
  }
  if (!reason) {
    notify("error", "请填写统一发布原因。");
    return;
  }
  const targets = records.value.filter((record) => selectedIds.value.includes(record.id));
  if (targets.some((record) => record.status === batchTarget.value)) {
    const same = targets
      .filter((record) => record.status === batchTarget.value)
      .map((record) => primaryText(record))
      .join("、");
    notify("error", `以下站点已是目标状态，无需发布：${same}`);
    return;
  }
  batchDraft.value = {
    id: crypto.randomUUID(),
    targetStatus: batchTarget.value,
    reason,
    stagedAt: new Date().toISOString(),
    snapshots: targets.map((record) => ({
      id: record.id,
      name: primaryText(record),
      beforeStatus: record.status
    }))
  };
  selectedIds.value = [];
  notify("success", `已暂存 ${targets.length} 个站点，提交时将再次校验当前状态。`);
}

function discardDraft() {
  batchDraft.value = null;
  notify("info", "已放弃暂存批次。");
}

function submitBatch() {
  const draft = batchDraft.value;
  if (!draft) return;
  const currentMap = new Map(records.value.map((record) => [record.id, record]));
  const conflicts = draft.snapshots.filter((snapshot) => {
    const current = currentMap.get(snapshot.id);
    return !current || current.status !== snapshot.beforeStatus;
  });
  if (conflicts.length > 0) {
    const detail = conflicts
      .map((snapshot) => {
        const current = currentMap.get(snapshot.id);
        return current
          ? `${snapshot.name}（暂存时 ${snapshot.beforeStatus}，当前 ${current.status}）`
          : `${snapshot.name}（已被删除）`;
      })
      .join("、");
    notify("error", `提交被拒绝：站点状态已偏离暂存前快照，整批未发布。${detail}`);
    return;
  }
  records.value = records.value.map((record) =>
    draft.snapshots.some((snapshot) => snapshot.id === record.id)
      ? { ...record, status: draft.targetStatus }
      : record
  );
  batchHistory.value = [
    {
      ...draft,
      publishedAt: new Date().toISOString(),
      undone: false
    },
    ...batchHistory.value
  ];
  const count = draft.snapshots.length;
  batchDraft.value = null;
  persist();
  notify("success", `批量发布成功，共更新 ${count} 个站点为「${draft.targetStatus}」。可在发布历史中撤回。`);
}

function laterBatchTouching(history: BatchHistoryEntry[], index: number, stationId: string) {
  for (let i = index - 1; i >= 0; i -= 1) {
    const entry = history[i];
    if (entry.undone) continue;
    if (entry.snapshots.some((snapshot) => snapshot.id === stationId)) return entry;
  }
  return undefined;
}

function undoBatch(entry: BatchHistoryEntry) {
  if (entry.undone) return;
  const index = batchHistory.value.findIndex((item) => item.id === entry.id);
  if (index < 0) return;
  const currentMap = new Map(records.value.map((record) => [record.id, record]));
  const conflicts = entry.snapshots
    .map((snapshot) => {
      const current = currentMap.get(snapshot.id);
      if (!current) return undefined;
      const later = laterBatchTouching(batchHistory.value, index, snapshot.id);
      if (later) return `${snapshot.name}（已被后续批次改动）`;
      if (current.status !== entry.targetStatus) {
        return `${snapshot.name}（当前 ${current.status}，非发布时的 ${entry.targetStatus}）`;
      }
      return undefined;
    })
    .filter((text): text is string => Boolean(text));
  if (conflicts.length > 0) {
    notify("error", `撤回被拒绝：存在已被后续批次改动的站点，本批未恢复。${conflicts.join("、")}`);
    return;
  }
  const missing = entry.snapshots.filter((snapshot) => !currentMap.has(snapshot.id));
  records.value = records.value.map((record) => {
    const snapshot = entry.snapshots.find((item) => item.id === record.id);
    return snapshot ? { ...record, status: snapshot.beforeStatus } : record;
  });
  batchHistory.value = batchHistory.value.map((item) =>
    item.id === entry.id ? { ...item, undone: true } : item
  );
  persist();
  notify(
    "success",
    `已撤回批次，恢复 ${entry.snapshots.length - missing.length} 个站点的原状态。` +
      (missing.length ? ` ${missing.length} 个站点已删除，跳过恢复。` : "")
  );
}

function formatTime(value: string) {
  return new Date(value).toLocaleString("zh-CN", { hour12: false });
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">{{ project.industry }}行业前端最小闭环</p>
          <h1>{{ project.title }}</h1>
          <p class="subtitle">{{ project.subtitle }}</p>
        </div>
        <div class="stack">
          <span v-for="item in project.stack" :key="item" class="tag">{{ item }}</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="(label, index) in project.metricLabels" :key="label" class="metric">
          <span>{{ label }}</span>
          <strong>{{ metrics[index] }}</strong>
        </article>
      </section>

      <section class="workspace">
        <form class="panel" @submit.prevent="submit">
          <h2>{{ project.formTitle }}</h2>
          <div class="form-grid">
            <label v-for="field in fields" :key="field.key">
              {{ field.label }}
              <select v-if="field.type === 'select'" v-model="form[field.key]" required>
                <option value="">请选择</option>
                <option v-for="option in field.options" :key="option">{{ option }}</option>
              </select>
              <input v-else v-model="form[field.key]" :type="field.type || 'text'" required />
            </label>
            <label>
              备注
              <textarea v-model="note" placeholder="填写处理说明或现场备注" />
            </label>
            <button type="submit">{{ project.primaryAction }}</button>
          </div>
        </form>

        <section class="list-panel">
          <div class="toolbar">
            <h2>{{ project.entityLabel }}列表</h2>
            <select v-model="filter">
              <option v-for="item in project.filters" :key="item">{{ item }}</option>
            </select>
          </div>

          <div v-if="message" class="batch-message" :class="message.type">{{ message.text }}</div>

          <div class="batch-bar">
            <div class="batch-row">
              <span class="batch-title">批量状态发布</span>
              <button type="button" :disabled="Boolean(batchDraft) || filteredRecords.length === 0" @click="selectFiltered">全选当前筛选</button>
              <button class="secondary" type="button" :disabled="selectedIds.length === 0" @click="clearSelection">清空勾选</button>
              <span class="batch-count">已勾选 {{ selectedIds.length }} 个站点</span>
            </div>
            <div class="batch-row">
              <select v-model="batchTarget" :disabled="Boolean(batchDraft)">
                <option v-for="item in statuses" :key="item" :value="item">发布为：{{ item }}</option>
              </select>
              <input
                v-model="batchReason"
                :disabled="Boolean(batchDraft)"
                placeholder="填写统一发布原因（必填）"
              />
              <button type="button" :disabled="Boolean(batchDraft)" @click="stageBatch">暂存批次</button>
            </div>
          </div>

          <section v-if="batchDraft" class="batch-draft">
            <div class="batch-draft-head">
              <strong>暂存批次（提交前保持快照校验）</strong>
              <span>{{ batchDraft.snapshots.length }} 站 → {{ batchDraft.targetStatus }}</span>
            </div>
            <p class="batch-reason">原因：{{ batchDraft.reason }}</p>
            <ul class="draft-list">
              <li v-for="snapshot in batchDraft.snapshots" :key="snapshot.id">
                <span>{{ snapshot.name }}</span>
                <span :class="{ deviated: draftCurrentStatus(snapshot.id) !== snapshot.beforeStatus }">
                  {{ snapshot.beforeStatus }}
                  <template v-if="draftCurrentStatus(snapshot.id) !== snapshot.beforeStatus">
                    → 已偏离：{{ draftCurrentStatus(snapshot.id) || "站点已删除" }}
                  </template>
                </span>
              </li>
            </ul>
            <div class="batch-row">
              <button type="button" @click="submitBatch">提交发布</button>
              <button class="secondary" type="button" @click="discardDraft">放弃暂存</button>
              <span class="batch-tip">任一站点状态偏离快照时整批拒绝，列表、指标和图表不受影响</span>
            </div>
          </section>

          <div class="record-grid">
            <div v-if="filteredRecords.length === 0" class="empty">暂无匹配数据</div>
            <article v-for="record in filteredRecords" :key="record.id" class="record">
              <div class="record-head">
                <label class="record-check">
                  <input
                    v-model="selectedIds"
                    type="checkbox"
                    :value="record.id"
                    :disabled="Boolean(batchDraft)"
                  />
                </label>
                <p class="record-title">{{ primaryText(record) }}</p>
                <span class="status">{{ record.status }}</span>
              </div>
              <div class="details">
                <span v-for="field in fields" :key="field.key">{{ field.label }}: {{ record[field.key] }}</span>
              </div>
              <p class="note">{{ record.notes }}</p>
              <div class="actions">
                <button type="button" @click="flow(record)">流转状态</button>
                <button class="secondary" type="button" @click="navigator.clipboard?.writeText(primaryText(record))">复制摘要</button>
                <button class="danger" type="button" @click="remove(record.id)">删除</button>
              </div>
            </article>
          </div>

          <div class="mini-chart">
            <div v-for="row in chartRows" :key="row.status" class="bar">
              <span>{{ row.status }}</span>
              <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
              <strong>{{ row.value }}</strong>
            </div>
          </div>

          <section class="batch-history">
            <h3>批量发布历史</h3>
            <div v-if="batchHistory.length === 0" class="empty">暂无批量发布记录</div>
            <article
              v-for="entry in batchHistory"
              :key="entry.id"
              class="history-item"
              :class="{ undone: entry.undone }"
            >
              <div class="history-head">
                <strong>{{ entry.snapshots.length }} 站 → {{ entry.targetStatus }}</strong>
                <span class="history-tag" :class="entry.undone ? 'tag-undone' : 'tag-live'">
                  {{ entry.undone ? "已撤回" : "可撤回" }}
                </span>
                <button
                  v-if="!entry.undone"
                  class="secondary"
                  type="button"
                  @click="undoBatch(entry)"
                >撤回批次</button>
              </div>
              <p class="batch-reason">原因：{{ entry.reason }}</p>
              <p class="history-meta">暂存 {{ formatTime(entry.stagedAt) }} · 发布 {{ formatTime(entry.publishedAt) }}</p>
              <ul class="draft-list">
                <li v-for="snapshot in entry.snapshots" :key="snapshot.id">
                  <span>{{ snapshot.name }}</span>
                  <span>{{ snapshot.beforeStatus }} → {{ entry.targetStatus }}</span>
                </li>
              </ul>
            </article>
          </section>
        </section>
      </section>
    </div>
  </main>
</template>
