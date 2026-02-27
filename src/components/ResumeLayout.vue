<script setup>
import { ref } from 'vue';

const props = defineProps({
  apiBase: String,
  accessToken: String,
});
const emit = defineEmits(['request-done']);

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

  if (!props.accessToken) {
    error.value = 'Сначала войдите в систему.';
    return;
  }
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
    <!-- HERO -->
    <section class="hero" id="how">
      <div class="hero-text">
        <h1>Сделай резюме сильнее за пару минут</h1>
        <p>
          Загрузите резюме и вакансию, а
          <span class="brand-inline">offerhelper</span>
          подскажет, чего не хватает и что улучшить, чтобы ближе подойти к офферу.
        </p>
        <div class="hero-badges">
          <span class="badge">Без регистрации</span>
          <span class="badge">AI‑рекомендации</span>
          <span class="badge">Drag &amp; Drop загрузка</span>
        </div>
      </div>
    </section>

    <!-- MAIN FORM -->
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

    <!-- ACTIONS -->
    <section class="actions">
      <button class="btn primary" @click="sendToAI" :disabled="loading">
        {{ loading ? 'Анализируем…' : 'Улучшить резюме под вакансию' }}
      </button>
      <button class="btn ghost" @click="clearAll">Очистить</button>
    </section>

    <!-- RESULT -->
    <section id="result" v-if="error || aiResult" class="result card">
      <h2>Рекомендации для улучшения</h2>
      <p v-if="error" class="error">{{ error }}</p>
      <pre v-if="aiResult" class="result-text">{{ aiResult }}</pre>
    </section>
  </div>
</template>

<style scoped>
/* сюда перенеси стили hero, grid, card, dropzone, textarea, actions, result из старого App.vue */
</style>
