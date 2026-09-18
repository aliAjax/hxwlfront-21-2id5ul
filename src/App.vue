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

type DraftStation = {
  id: string;
  name: string;
  snapshot: string;
};

type BatchDraft = {
  stations: DraftStation[];
  targetStatus: string;
  reason: string;
  stagedAt: string;
};

type BatchHistoryEntry = {
  id: string;
  reason: string;
  targetStatus: string;
  publishedAt: string;
  changes: { id: string; name: string; before: string; after: string }[];
  withdrawn: boolean;
  withdrawnAt?: string;
};

type Notice = {
  type: "success" | "error" | "info";
  text: string;
};

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
  "draftKey": "hxwlfront-21-batch-draft",
  "historyKey": "hxwlfront-21-batch-history",
  "filterKey": "hxwlfront-21-filter",
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

function loadDraft(): BatchDraft | null {
  const raw = localStorage.getItem(project.draftKey);
  if (!raw) return null;
  try {
    const parsed = JSON.parse(raw) as BatchDraft;
    if (!Array.isArray(parsed.stations) || typeof parsed.targetStatus !== "string" || typeof parsed.reason !== "string") {
      return null;
    }
    return parsed;
  } catch {
    return null;
  }
}

function loadHistory(): BatchHistoryEntry[] {
  const raw = localStorage.getItem(project.historyKey);
  if (!raw) return [];
  try {
    const parsed = JSON.parse(raw) as BatchHistoryEntry[];
    return Array.isArray(parsed) ? parsed : [];
  } catch {
    return [];
  }
}

function loadFilter(): string {
  const saved = localStorage.getItem(project.filterKey);
  return saved && project.filters.includes(saved as (typeof project.filters)[number]) ? saved : project.filters[0];
}

const records = ref<RecordItem[]>(loadRecords());
const form = reactive<Record<string, string | number>>(createBlank());
const note = ref("");
const filter = ref(loadFilter());
const draft = ref<BatchDraft | null>(loadDraft());
const history = ref<BatchHistoryEntry[]>(loadHistory());
const selectedIds = ref<string[]>([]);
const targetStatus = ref<string>(draft.value?.targetStatus ?? "");
const reason = ref<string>(draft.value?.reason ?? "");
const notice = ref<Notice | null>(null);

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
  localStorage.setItem(project.draftKey, draft.value ? JSON.stringify(draft.value) : "");
  localStorage.setItem(project.historyKey, JSON.stringify(history.value));
}

watch(filter, (value) => {
  localStorage.setItem(project.filterKey, value);
});

const draftIdSet = computed(() => new Set((draft.value?.stations ?? []).map((station) => station.id)));

const visibleChecked = computed({
  get: () => filteredRecords.value.length > 0 && filteredRecords.value.every((record) => selectedIds.value.includes(record.id)),
  set: (checked: boolean) => {
    const visibleIds = filteredRecords.value.map((record) => record.id);
    if (checked) {
      selectedIds.value = [...new Set([...selectedIds.value, ...visibleIds])];
    } else {
      const visibleSet = new Set(visibleIds);
      selectedIds.value = selectedIds.value.filter((id) => !visibleSet.has(id));
    }
  }
});

const visibleIndeterminate = computed(
  () => !visibleChecked.value && filteredRecords.value.some((record) => selectedIds.value.includes(record.id))
);

function isSelected(id: string) {
  return selectedIds.value.includes(id);
}

function toggleSelected(id: string) {
  if (isSelected(id)) {
    selectedIds.value = selectedIds.value.filter((item) => item !== id);
  } else {
    selectedIds.value = [...selectedIds.value, id];
  }
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

function formatTime(iso: string) {
  const date = new Date(iso);
  if (Number.isNaN(date.getTime())) return iso;
  const pad = (value: number) => String(value).padStart(2, "0");
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}`;
}

function shortId(id: string) {
  return id.slice(0, 8);
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
  selectedIds.value = selectedIds.value.filter((item) => item !== id);
  persist();
}

function stageBatch() {
  if (!targetStatus.value) {
    notice.value = { type: "error", text: "请选择要统一发布的目标状态。" };
    return;
  }
  if (!reason.value.trim()) {
    notice.value = { type: "error", text: "请填写统一原因后再暂存。" };
    return;
  }
  if (selectedIds.value.length === 0) {
    notice.value = { type: "error", text: "请先勾选需要批量发布的站点。" };
    return;
  }
  const stations: DraftStation[] = selectedIds.value.map((id) => {
    const record = records.value.find((item) => item.id === id);
    return {
      id,
      name: record ? primaryText(record) : id,
      snapshot: record?.status ?? ""
    };
  });
  draft.value = {
    stations,
    targetStatus: targetStatus.value,
    reason: reason.value.trim(),
    stagedAt: new Date().toISOString()
  };
  selectedIds.value = [];
  persist();
  notice.value = {
    type: "success",
    text: `已暂存 ${stations.length} 个站点（目标：${targetStatus.value}），刷新页面后仍保留，可直接提交或放弃。`
  };
}

function submitDraft() {
  if (!draft.value) return;
  const current = draft.value;
  const conflicts = current.stations.filter((station) => {
    const record = records.value.find((item) => item.id === station.id);
    return !record || record.status !== station.snapshot;
  });
  if (conflicts.length > 0) {
    notice.value = {
      type: "error",
      text: `整批拒绝：${conflicts.map((station) => station.name).join("、")} 的当前状态已偏离暂存前快照，本次未发布任何站点。请重新勾选暂存后再提交。`
    };
    return;
  }
  const changedAt = new Date().toISOString();
  const changes = current.stations.map((station) => ({
    id: station.id,
    name: station.name,
    before: station.snapshot,
    after: current.targetStatus
  }));
  records.value = records.value.map((record) =>
    draftIdSet.value.has(record.id) ? { ...record, status: current.targetStatus } : record
  );
  history.value = [
    {
      id: crypto.randomUUID(),
      reason: current.reason,
      targetStatus: current.targetStatus,
      publishedAt: changedAt,
      changes,
      withdrawn: false
    },
    ...history.value
  ];
  draft.value = null;
  targetStatus.value = "";
  reason.value = "";
  selectedIds.value = [];
  persist();
  notice.value = {
    type: "success",
    text: `批量发布成功，共更新 ${changes.length} 个站点为「${current.targetStatus}」，前后状态已记入历史，可在历史中撤回整批。`
  };
}

function discardDraft() {
  draft.value = null;
  targetStatus.value = "";
  reason.value = "";
  selectedIds.value = [];
  persist();
  notice.value = { type: "info", text: "已放弃暂存的批量发布，站点状态未做任何修改。" };
}

function withdrawBatch(entry: BatchHistoryEntry) {
  if (entry.withdrawn) return;
  const current = records.value;
  const conflictReasons: string[] = [];
  const deleted: string[] = [];
  for (const change of entry.changes) {
    const later = history.value.find(
      (item) =>
        !item.withdrawn &&
        item.publishedAt > entry.publishedAt &&
        item.changes.some((stationChange) => stationChange.id === change.id)
    );
    if (later) {
      conflictReasons.push(`${change.name} 已被后续批次「${later.reason}」改动`);
      continue;
    }
    const record = current.find((item) => item.id === change.id);
    if (!record) {
      deleted.push(change.name);
    } else if (record.status !== change.after) {
      conflictReasons.push(`${change.name} 当前状态为「${record.status}」，已偏离发布后状态「${change.after}」`);
    }
  }
  if (conflictReasons.length > 0) {
    notice.value = {
      type: "error",
      text: `拒绝恢复：${conflictReasons.join("；")}。该批次未执行任何恢复。`
    };
    return;
  }
  const idSet = new Set(entry.changes.map((change) => change.id));
  records.value = records.value.map((record) => {
    const change = entry.changes.find((item) => item.id === record.id);
    return change && idSet.has(record.id) ? { ...record, status: change.before } : record;
  });
  entry.withdrawn = true;
  entry.withdrawnAt = new Date().toISOString();
  persist();
  notice.value = {
    type: "success",
    text: `已撤回批次「${entry.reason}」，${entry.changes.length - deleted.length} 个站点恢复为发布前状态。`
      + (deleted.length ? `（${deleted.join("、")} 已删除，未恢复）` : "")
  };
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

          <section class="batch-panel">
            <h3>批量状态发布</h3>
            <p v-if="notice" class="notice" :class="notice.type">{{ notice.text }}</p>
            <template v-if="draft">
              <p class="draft-info">
                暂存于 {{ formatTime(draft.stagedAt) }}：共 {{ draft.stations.length }} 个站点，
                目标「{{ draft.targetStatus }}」，原因：{{ draft.reason }}
              </p>
              <ul class="draft-stations">
                <li v-for="station in draft.stations" :key="station.id">
                  {{ station.name }}：{{ station.snapshot }} → {{ draft.targetStatus }}
                </li>
              </ul>
              <div class="actions">
                <button type="button" @click="submitDraft">检查快照并提交发布</button>
                <button class="secondary" type="button" @click="discardDraft">放弃暂存</button>
              </div>
            </template>
            <template v-else>
              <div class="batch-row">
                <label class="check-cell">
                  <input
                    type="checkbox"
                    class="check-box"
                    :checked="visibleChecked"
                    :indeterminate.prop="visibleIndeterminate"
                    @change="visibleChecked = ($event.target as HTMLInputElement).checked"
                  />
                  <span>全选当前筛选</span>
                </label>
                <label>
                  目标状态
                  <select v-model="targetStatus">
                    <option value="">请选择</option>
                    <option v-for="item in statuses" :key="item">{{ item }}</option>
                  </select>
                </label>
                <label class="reason-cell">
                  统一原因
                  <input v-model="reason" placeholder="如：片区检修，统一暂停营业" />
                </label>
              </div>
              <div class="actions">
                <button type="button" @click="stageBatch">暂存勾选站点（{{ selectedIds.length }}）</button>
              </div>
            </template>
          </section>

          <div class="record-grid">
            <div v-if="filteredRecords.length === 0" class="empty">暂无匹配数据</div>
            <article v-for="record in filteredRecords" :key="record.id" class="record">
              <div class="record-head">
                <p class="record-title">
                  <label class="check-cell">
                    <input
                      type="checkbox"
                      class="check-box"
                      :checked="isSelected(record.id)"
                      :disabled="!!draft"
                      @change="toggleSelected(record.id)"
                    />
                    <span>{{ primaryText(record) }}</span>
                  </label>
                  <span v-if="draftIdSet.has(record.id)" class="staged-flag">已暂存</span>
                </p>
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

          <section v-if="history.length" class="history">
            <h3>批量发布历史</h3>
            <div v-for="entry in history" :key="entry.id" class="history-item" :class="{ withdrawn: entry.withdrawn }">
              <div class="history-head">
                <div>
                  <strong>批次 #{{ shortId(entry.id) }}</strong>
                  <span class="history-meta">{{ formatTime(entry.publishedAt) }} · {{ entry.reason }} · {{ entry.changes.length }} 站 → {{ entry.targetStatus }}</span>
                </div>
                <button
                  v-if="!entry.withdrawn"
                  class="secondary"
                  type="button"
                  @click="withdrawBatch(entry)"
                >撤回整批</button>
                <span v-else class="withdrawn-flag">
                  已撤回<span v-if="entry.withdrawnAt"> · {{ formatTime(entry.withdrawnAt) }}</span>
                </span>
              </div>
              <p class="history-changes">
                <template v-for="(change, index) in entry.changes" :key="change.id">
                  {{ change.name }}：{{ change.before }} → {{ change.after }}<template v-if="index < entry.changes.length - 1">；</template>
                </template>
              </p>
            </div>
          </section>

          <div class="mini-chart">
            <div v-for="row in chartRows" :key="row.status" class="bar">
              <span>{{ row.status }}</span>
              <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
              <strong>{{ row.value }}</strong>
            </div>
          </div>
        </section>
      </section>
    </div>
  </main>
</template>
