<script setup>
import { ref } from 'vue';

const props = defineProps({
  requests: { type: Array, default: () => [] },
  error: String,
});
const emit = defineEmits(['refresh']);

const selected = ref(null);
function select(r) {
  selected.value = r;
}
</script>

<template>
  <aside class="history-sidebar card">
    <div class="sidebar-header">
      <h2>История</h2>
      <button class="btn ghost btn-xs" type="button" @click="emit('refresh')">
        Обновить
      </button>
    </div>
    <p v-if="error" class="error">{{ error }}</p>
    <div v-if="requests.length" class="sidebar-list">
      <button
        v-for="r in requests"
        :key="r.id"
        type="button"
        class="sidebar-item"
        :class="{ 'sidebar-item--active': selected && selected.id === r.id }"
        @click="select(r)"
      >
        <div class="sidebar-date">
          {{ new Date(r.created_at).toLocaleString() }}
        </div>
        <div class="sidebar-prompt">
          {{ r.prompt || 'generate-resume' }}
        </div>
      </button>
    </div>
    <p v-else class="empty">Пока нет запросов</p>

    <div v-if="selected" class="sidebar-answer">
      <h3>Ответ</h3>
      <div class="sidebar-answer-box">
        <pre>{{ selected.response || 'Ответ пустой' }}</pre>
      </div>
    </div>
  </aside>
</template>

<style scoped>
.history-sidebar {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.card {
  background: #ffffff;
  border-radius: 14px;
  padding: 12px 14px 14px;
  box-shadow: 0 10px 26px rgba(15, 23, 42, 0.06);
  border: 1px solid #e5e7eb;
}
.sidebar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.sidebar-list {
  max-height: 280px;
  overflow-y: auto;
  border-radius: 10px;
  border: 1px solid #e5e7eb;
  background: #f9fafb;
}
.sidebar-item {
  width: 100%;
  text-align: left;
  padding: 8px 10px;
  border: none;
  background: transparent;
  cursor: pointer;
  border-bottom: 1px solid #e5e7eb;
}
.sidebar-item--active {
  background: #eef2ff;
}
.sidebar-item:last-child {
  border-bottom: none;
}
.sidebar-date {
  font-size: 0.8rem;
  color: #6b7280;
}
.sidebar-prompt {
  font-size: 0.9rem;
  color: #374151;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.sidebar-answer {
  margin-top: 8px;
}
.sidebar-answer-box {
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  background: #ffffff;
  padding: 6px 8px;
  max-height: 200px;
  overflow: auto;
}
.sidebar-answer-box pre {
  margin: 0;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    'Liberation Mono', 'Courier New', monospace;
  font-size: 0.85rem;
  white-space: pre-wrap;
  word-wrap: break-word;
}
.btn-xs {
  padding: 4px 10px;
  font-size: 0.8rem;
}
.btn.ghost {
  background: transparent;
  border-color: #d1d5db;
  color: #374151;
  border-radius: 999px;
  border-width: 1px;
}
.error {
  color: #b91c1c;
}
.empty {
  font-size: 0.9rem;
  color: #6b7280;
}
</style>
