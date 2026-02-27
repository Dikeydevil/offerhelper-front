<script setup>
import { ref, computed } from 'vue';

const props = defineProps({
  apiBase: String,
  accessToken: [String, Object],
});
const emit = defineEmits(['request-done']);

const isAuthed = computed(() => !!props.accessToken);

const resumeText = ref('');
const vacancyText = ref('');
const resumeFile = ref(null);
const vacancyFile = ref(null);

const resumeDragOver = ref(false);
const vacancyDragOver = ref(false);

const aiResult = ref('');
const loading = ref(false);
const error = ref('');

function handleDrop(e, type) {
  const file = e.dataTransfer.files[0];
  if (!file) return;
  if (type === 'resume') {
    resumeFile.value = file;
    resumeDragOver.value = false;
  } else {
    vacancyFile.value = file;
    vacancyDragOver.value = false;
  }
}

function handleFileInput(e, type) {
  const file = e.target.files[0];
  if (!file) return;
  if (type === 'resume') resumeFile.value = file;
  else vacancyFile.value = file;
}

function clearAll() {
  resumeText.value = '';
  vacancyText.value = '';
  resumeFile.value = null;
  vacancyFile.value = null;
  aiResult.value = '';
  error.value = '';
}

async function sendToAI() {
  error.value = '';
  aiResult.value = '';

  if (!resumeFile.value && !resumeText.value.trim()) {
    error.value = 'Заполните резюме: файл или текст.';
    return;
  }
  if (!vacancyFile.value && !vacancyText.value.trim()) {
    error.value = 'Заполните вакансию: файл или текст.';
    return;
  }

  loading.value = true;
  try {
    const formData = new FormData();
    if (resumeFile.value) formData.append('resume', resumeFile.value);
    if (vacancyFile.value) formData.append('vacancy', vacancyFile.value);
    if (resumeText.value.trim()) formData.append('resume_text', resumeText.value.trim());
    if (vacancyText.value.trim()) formData.append('vacancy_text', vacancyText.value.trim());

    const res = await fetch(`${props.apiBase}/gigachat/generate-resume`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${props.accessToken}`,
      },
      body: formData,
    });

    if (!res.ok) throw new Error((await res.text()) || 'Ошибка при обращении к нейросети.');

    const data = await res.json();
    aiResult.value = data.resume_text || 'Ответ получен, но резюме пустое.';
    emit('request-done');
  } catch (e) {
    error.value = e.message || 'Ошибка при обращении к нейросети.';
  } finally {
    loading.value = false;
  }
}
</script>

<template>
  <div class="resume-layout">
    <!-- HERO: ВСЕГДА -->
    <section class="hero" id="how">
      <div class="hero-text">
        <h1>Сделай резюме сильнее за пару минут</h1>
        <p>
          Загрузите резюме и вакансию, а
          <span class="brand-inline">offerhelper</span>
          подскажет, чего не хватает и что улучшить, чтобы ближе подойти к офферу.
        </p>
        <div class="hero-badges">
          <span class="badge">AI‑рекомендации</span>
          <span class="badge">Drag &amp; Drop загрузка</span>
        </div>
      </div>
    </section>

    <!-- ВСЁ НИЖЕ ТОЛЬКО ДЛЯ АВТОРИЗОВАННОГО -->
    <div v-if="isAuthed">
      <main class="grid">
        <section class="card">
          <h2>Резюме</h2>
          <div
            class="dropzone"
            :class="{ 'dropzone--over': resumeDragOver }"
            @dragover.prevent="resumeDragOver = true"
            @dragleave.prevent="resumeDragOver = false"
            @drop.prevent="handleDrop($event, 'resume')"
          >
            <p class="dropzone-title">Перетащите файл с резюме сюда</p>
            <p class="dropzone-sub">Поддерживаются изображения, PDF, DOC/DOCX</p>
            <input
              type="file"
              accept="image/*,.pdf,.doc,.docx,.txt"
              @change="(e) => handleFileInput(e, 'resume')"
            />
            <p v-if="resumeFile" class="file-label">Файл: {{ resumeFile.name }}</p>
          </div>
          <label class="label">
            Или вставьте текст:
            <textarea
              v-model="resumeText"
              rows="8"
              placeholder="Вставьте сюда текст вашего резюме"
            />
          </label>
        </section>

        <section class="card">
          <h2>Вакансия</h2>
          <div
            class="dropzone"
            :class="{ 'dropzone--over': vacancyDragOver }"
            @dragover.prevent="vacancyDragOver = true"
            @dragleave.prevent="vacancyDragOver = false"
            @drop.prevent="handleDrop($event, 'vacancy')"
          >
            <p class="dropzone-title">Перетащите скрин или файл с вакансией</p>
            <p class="dropzone-sub">Например, скриншот из hh.ru или PDF‑описание</p>
            <input
              type="file"
              accept="image/*,.pdf"
              @change="(e) => handleFileInput(e, 'vacancy')"
            />
            <p v-if="vacancyFile" class="file-label">Файл: {{ vacancyFile.name }}</p>
          </div>
          <label class="label">
            Или вставьте текст:
            <textarea
              v-model="vacancyText"
              rows="8"
              placeholder="Вставьте сюда текст вакансии"
            />
          </label>
        </section>
      </main>

      <section class="actions">
        <button class="btn primary" @click="sendToAI" :disabled="loading">
          {{ loading ? 'Анализируем…' : 'Улучшить резюме под вакансию' }}
        </button>
        <button class="btn ghost" @click="clearAll">Очистить</button>
      </section>

      <section id="result" v-if="error || aiResult" class="result card">
        <h2>Рекомендации для улучшения</h2>
        <p v-if="error" class="error">{{ error }}</p>
        <pre v-if="aiResult" class="result-text">{{ aiResult }}</pre>
      </section>
    </div>
  </div>
</template>

<style scoped>
.resume-layout {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

/* HERO */
.hero {
  margin: 8px 0 8px;
  padding: 22px 20px 20px;
  border-radius: 20px;
  background: radial-gradient(circle at top left, #dbeafe, #f5f3ff 45%, #f9fafb);
  border: 1px solid #e5e7eb;
}
.hero-text h1 {
  font-size: clamp(1.8rem, 3vw, 2.4rem);
  margin-bottom: 6px;
  color: #0f172a;
}
.hero-text p {
  max-width: 640px;
  color: #4b5563;
  margin-bottom: 8px;
}
.brand-inline {
  font-weight: 600;
  color: #4f46e5;
}
.hero-badges {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
.badge {
  padding: 3px 9px;
  border-radius: 999px;
  background: #eef2ff;
  color: #4338ca;
  font-size: 0.78rem;
}

/* GRID / CARDS */
.grid {
  display: grid;
  gap: 16px;
}
@media (min-width: 900px) {
  .grid {
    grid-template-columns: 1fr 1fr;
  }
}
.card {
  background: #ffffff;
  border-radius: 14px;
  padding: 16px 18px 18px;
  box-shadow: 0 10px 26px rgba(15, 23, 42, 0.06);
  border: 1px solid #e5e7eb;
}
.card h2 {
  margin-bottom: 10px;
}

/* DROPZONE */
.dropzone {
  border: 1.5px dashed #c7d2fe;
  border-radius: 12px;
  padding: 16px 14px 14px;
  text-align: center;
  color: #6b7280;
  background: #f9fafb;
  transition: 0.15s ease;
  margin-bottom: 10px;
}
.dropzone--over {
  border-color: #4f46e5;
  background: #eef2ff;
  color: #3730a3;
}
.dropzone-title {
  font-weight: 500;
}
.dropzone-sub {
  font-size: 0.8rem;
  margin-top: 4px;
}
.dropzone input[type='file'] {
  margin-top: 8px;
}
.file-label {
  margin-top: 6px;
  font-size: 0.85rem;
  color: #374151;
}

/* FORM */
.label {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 0.9rem;
}
textarea {
  resize: vertical;
  min-height: 120px;
  padding: 8px 10px;
  border-radius: 10px;
  border: 1px solid #d1d5db;
  font: inherit;
  line-height: 1.35;
}
textarea:focus {
  outline: none;
  border-color: #4f46e5;
  box-shadow: 0 0 0 1px rgba(79, 70, 229, 0.15);
}

/* ACTIONS */
.actions {
  margin: 16px 0 10px;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}
.btn {
  border-radius: 999px;
  padding: 8px 16px;
  font: inherit;
  cursor: pointer;
  border: 1px solid transparent;
}
.btn.primary {
  background: linear-gradient(135deg, #4f46e5, #6366f1);
  color: #ffffff;
  box-shadow: 0 8px 18px rgba(79, 70, 229, 0.35);
}
.btn.primary:disabled {
  opacity: 0.7;
  cursor: default;
  box-shadow: none;
}
.btn.ghost {
  background: transparent;
  border-color: #d1d5db;
  color: #374151;
}

/* RESULT */
.result {
  margin-top: 4px;
}
.result-text {
  white-space: pre-wrap;
  margin-top: 6px;
  font-size: 0.94rem;
}
.error {
  color: #b91c1c;
}
</style>
