<script setup>
import { ref, computed, onMounted } from 'vue';

const API_BASE = 'http://127.0.0.1:8000';

// ---------- AUTH ----------
const email = ref('');
const password = ref('');
const accessToken = ref(null);
const authError = ref('');
const me = ref(null);

const isAuthed = computed(() => !!accessToken.value);

// выбранный запрос из истории
const selectedRequest = ref(null);

function selectRequest(r) {
  if (selectedRequest.value && selectedRequest.value.id === r.id) {
    selectedRequest.value = null;
  } else {
    selectedRequest.value = r;
  }
}

function setToken(token) {
  accessToken.value = token;
  localStorage.setItem('access_token', token);
}

function clearToken() {
  accessToken.value = null;
  localStorage.removeItem('access_token');
  me.value = null;
}

async function fetchMe() {
  if (!accessToken.value) return;
  try {
    const res = await fetch(`${API_BASE}/me`, {
      headers: { Authorization: `Bearer ${accessToken.value}` },
    });
    if (!res.ok) throw new Error('Не удалось получить профиль');
    me.value = await res.json();
  } catch (e) {
    clearToken();
  }
}

onMounted(async () => {
  const saved = localStorage.getItem('access_token');
  if (saved) {
    accessToken.value = saved;
    await fetchMe();
    await loadRequests();
  }
});

async function register() {
  authError.value = '';
  try {
    const res = await fetch(`${API_BASE}/auth/register`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email: email.value, password: password.value }),
    });

    if (!res.ok) {
      const text = await res.text();
      throw new Error(text || 'Ошибка регистрации');
    }

    await login();
  } catch (e) {
    authError.value = e.message || 'Ошибка регистрации';
  }
}

async function login() {
  authError.value = '';
  try {
    const body = new URLSearchParams();
    body.append('username', email.value);
    body.append('password', password.value);

    const res = await fetch(`${API_BASE}/auth/login`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body,
    });

    if (!res.ok) {
      const text = await res.text();
      throw new Error(text || 'Ошибка входа');
    }

    const data = await res.json();
    setToken(data.access_token);
    await fetchMe();
    await loadRequests();
  } catch (e) {
    authError.value = e.message || 'Ошибка входа';
  }
}

function logout() {
  clearToken();
}

// ---------- RESUME FORM ----------
const resumeText = ref('');
const vacancyText = ref('');
const resumeFile = ref(null);
const vacancyFile = ref(null);

const resumeDragOver = ref(false);
const vacancyDragOver = ref(false);

const aiResult = ref('');
const loading = ref(false);
const error = ref('');

// история запросов
const requests = ref([]);
const requestsError = ref('');

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

async function loadRequests() {
  requestsError.value = '';
  if (!accessToken.value) return;
  try {
    const res = await fetch(`${API_BASE}/me/requests`, {
      headers: { Authorization: `Bearer ${accessToken.value}` },
    });
    if (!res.ok) throw new Error('Не удалось загрузить историю запросов');
    requests.value = await res.json();
  } catch (e) {
    requestsError.value = e.message || 'Ошибка загрузки истории';
  }
}

async function sendToAI() {
  error.value = '';
  aiResult.value = '';

  if (!accessToken.value) {
    error.value = 'Сначала войдите в систему.';
    return;
  }

  // хотя бы что‑то одно должно быть
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

    // файлы, если есть
    if (resumeFile.value) {
      formData.append('resume', resumeFile.value);
    }
    if (vacancyFile.value) {
      formData.append('vacancy', vacancyFile.value);
    }

    // текст, если есть
    if (resumeText.value.trim()) {
      formData.append('resume_text', resumeText.value.trim());
    }
    if (vacancyText.value.trim()) {
      formData.append('vacancy_text', vacancyText.value.trim());
    }

    const res = await fetch(`${API_BASE}/gigachat/generate-resume`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${accessToken.value}`,
      },
      body: formData,
    });

    if (!res.ok) {
      const text = await res.text();
      throw new Error(text || 'Ошибка при обращении к нейросети.');
    }

    const data = await res.json();
    aiResult.value = data.resume_text || 'Ответ получен, но резюме пустое.';
    await loadRequests();
  } catch (e) {
    error.value = e.message || 'Ошибка при обращении к нейросети.';
  } finally {
    loading.value = false;
  }
}

</script>

<template>
  <div class="page">
    <!-- HEADER -->
    <header class="site-header">
      <div class="site-header__inner">
        <div class="brand">
          <div class="brand-mark">oh</div>
          <span class="brand-name">offerhelper</span>
        </div>
        <nav class="nav">
          <a href="#how" class="nav-link">Как это работает</a>
          <a href="#result" class="nav-link">Результат</a>
        </nav>
      </div>
    </header>

    <!-- AUTH -->
    <section class="auth card">
      <h2>Вход</h2>
      <p v-if="me">Вы вошли как {{ me.email }}</p>
      <p v-if="authError" class="error">{{ authError }}</p>

      <div v-if="!isAuthed">
        <input
          v-model="email"
          type="email"
          placeholder="Email"
          class="auth-input"
        />
        <input
          v-model="password"
          type="password"
          placeholder="Пароль"
          class="auth-input"
        />
        <div class="auth-actions">
          <button class="btn primary" @click="login">Войти</button>
          <button class="btn ghost" @click="register">Регистрация</button>
        </div>
      </div>

      <div v-else>
        <button class="btn ghost" @click="logout">Выйти</button>
      </div>
    </section>

    <!-- HERO -->
    <section class="hero" id="how">
      <div class="hero-text">
        <h1>Сделай резюме сильнее за пару минут</h1>
        <p>
          Загрузите резюме и вакансию, а
          <span class="brand-inline">offerhelper</span>
          подскажет, чего не хватает и что улучшить, чтобы ближе подойти к
          офферу.
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
      <!-- Резюме -->
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
          <p v-if="resumeFile" class="file-label">
            Файл: {{ resumeFile.name }}
          </p>
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

      <!-- Вакансия -->
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
          <p class="dropzone-sub">
            Например, скриншот из hh.ru или PDF‑описание
          </p>
          <input
            type="file"
            accept="image/*,.pdf"
            @change="(e) => handleFileInput(e, 'vacancy')"
          />
          <p v-if="vacancyFile" class="file-label">
            Файл: {{ vacancyFile.name }}
          </p>
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

    <!-- HISTORY: список + только ответ -->
    <section v-if="isAuthed" class="card history">
      <div class="history-header">
        <h2>История запросов</h2>
        <button
          class="btn ghost history-refresh"
          type="button"
          @click="loadRequests"
        >
          Обновить
        </button>
      </div>

      <p v-if="requestsError" class="error">{{ requestsError }}</p>

      <div v-if="requests.length" class="history-layout">
        <ul class="requests-list">
          <li
            v-for="r in requests"
            :key="r.id"
            class="request-item"
            :class="{ 'request-item--active': selectedRequest && selectedRequest.id === r.id }"
          >
            <button
              type="button"
              class="request-button"
              @click="selectRequest(r)"
            >
              <div class="request-main-line">
                <span class="request-date">
                  {{ new Date(r.created_at).toLocaleString() }}
                </span>
              </div>
              <div class="request-prompt">
                {{ r.prompt || 'generate-resume' }}
              </div>
            </button>
          </li>
        </ul>

        <div class="request-answer">
          <h3 v-if="selectedRequest">Ответ</h3>
          <p v-else class="request-answer-placeholder">
            Выберите запрос слева, чтобы посмотреть ответ.
          </p>

          <div v-if="selectedRequest" class="request-answer-box">
            <pre>{{ selectedRequest.response || 'Ответ пустой' }}</pre>
          </div>
        </div>
      </div>

      <p v-else>Пока нет запросов</p>
    </section>

    <footer class="footer">
      <small>offerhelper · прототип AI‑сервиса улучшения резюме</small>
    </footer>
  </div>
</template>

<style scoped>
:root {
  color-scheme: light dark;
}
.page {
  max-width: 1120px;
  margin: 0 auto;
  padding: 16px 16px 40px;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI',
    sans-serif;
  color: #0f172a;
}

/* HEADER */
.site-header {
  position: sticky;
  top: 0;
  z-index: 20;
  backdrop-filter: blur(14px);
  background: linear-gradient(
    to right,
    rgba(15, 23, 42, 0.85),
    rgba(30, 64, 175, 0.85)
  );
  border-radius: 999px;
  margin: 4px auto 18px;
  max-width: 1120px;
  box-shadow: 0 14px 35px rgba(15, 23, 42, 0.45);
}
.site-header__inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 18px;
  color: #e5e7eb;
}
.brand {
  display: flex;
  align-items: center;
  gap: 10px;
}
.brand-mark {
  width: 28px;
  height: 28px;
  border-radius: 9px;
  background: linear-gradient(135deg, #a855f7, #22c55e);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 14px;
  color: #0b1020;
}
.brand-name {
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: lowercase;
}
.nav {
  display: flex;
  gap: 12px;
  font-size: 0.9rem;
}
.nav-link {
  color: #e5e7eb;
  text-decoration: none;
  padding: 4px 10px;
  border-radius: 999px;
  transition: background 0.15s ease, color 0.15s ease;
}
.nav-link:hover {
  background: rgba(15, 23, 42, 0.8);
  color: #f9fafb;
}

/* AUTH */
.auth {
  margin-bottom: 16px;
}
.auth-input {
  display: block;
  width: 100%;
  margin-bottom: 8px;
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid #d1d5db;
}
.auth-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

/* HERO */
.hero {
  margin: 18px 0 18px;
  padding: 26px 20px 24px;
  border-radius: 20px;
  background: radial-gradient(
    circle at top left,
    #e0f2fe,
    #f5f3ff 45%,
    #f9fafb
  );
  border: 1px solid #e5e7eb;
}
.hero-text h1 {
  font-size: clamp(1.7rem, 3vw, 2.3rem);
  margin-bottom: 8px;
  color: #0f172a;
}
.hero-text p {
  max-width: 640px;
  color: #4b5563;
  margin-bottom: 10px;
}
.brand-inline {
  font-weight: 600;
  color: #4f46e5;
}
.hero-badges {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.badge {
  padding: 4px 10px;
  border-radius: 999px;
  background: #eef2ff;
  color: #4338ca;
  font-size: 0.8rem;
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
  border: 2px dashed #c7d2fe;
  border-radius: 12px;
  padding: 16px 14px 14px;
  text-align: center;
  color: #6b7280;
  background: #f9fafb;
  transition: 0.15s ease;
  margin-bottom: 12px;
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
  margin-top: 10px;
}
.file-label {
  margin-top: 8px;
  font-size: 0.9rem;
  color: #374151;
}

/* FORM */
.label {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 0.95rem;
}
textarea {
  resize: vertical;
  min-height: 120px;
  padding: 10px 12px;
  border-radius: 10px;
  border: 1px solid #d1d5db;
  font: inherit;
}
textarea:focus {
  outline: none;
  border-color: #4f46e5;
  box-shadow: 0 0 0 1px rgba(79, 70, 229, 0.15);
}

/* ACTIONS */
.actions {
  margin: 20px 0 12px;
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}
.btn {
  border-radius: 999px;
  padding: 9px 18px;
  font: inherit;
  cursor: pointer;
  border: 1px solid transparent;
}
.btn.primary {
  background: linear-gradient(135deg, #4f46e5, #6366f1);
  color: #ffffff;
}
.btn.primary:disabled {
  opacity: 0.7;
  cursor: default;
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
  margin-top: 8px;
  font-size: 0.95rem;
}
.error {
  color: #b91c1c;
}

/* FOOTER */
.footer {
  margin-top: 20px;
  text-align: center;
  color: #9ca3af;
  font-size: 0.85rem;
}

/* HISTORY */
.history {
  margin-top: 16px;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}

.history-refresh {
  padding-inline: 12px;
}

.history-layout {
  display: grid;
  grid-template-columns: minmax(0, 260px) minmax(0, 1fr);
  gap: 12px;
  margin-top: 10px;
}

@media (max-width: 800px) {
  .history-layout {
    grid-template-columns: minmax(0, 1fr);
  }
}

.requests-list {
  list-style: none;
  padding: 0;
  margin: 0;
  max-height: 260px;
  overflow-y: auto;
  border-radius: 10px;
  border: 1px solid #e5e7eb;
  background: #f9fafb;
}

.request-item + .request-item {
  border-top: 1px solid #e5e7eb;
}

.request-button {
  display: block;
  width: 100%;
  text-align: left;
  padding: 8px 10px;
  background: transparent;
  border: none;
  cursor: pointer;
}

.request-item--active .request-button {
  background: #eef2ff;
}

.request-item--active {
  border-color: #4f46e5;
}

.request-main-line {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  margin-bottom: 4px;
}

.request-date {
  color: #6b7280;
}

.request-prompt {
  font-size: 0.9rem;
  color: #374151;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* блок с ответом */
.request-answer {
  border-radius: 10px;
  border: 1px solid #e5e7eb;
  padding: 10px 12px;
  background: #f9fafb;
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 260px;
}

.request-answer-placeholder {
  margin: 0;
  color: #6b7280;
}

.request-answer-box {
  flex: 1;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  background: #ffffff;
  padding: 8px 10px;
  overflow: auto;
}

.request-answer-box pre {
  margin: 0;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    'Liberation Mono', 'Courier New', monospace;
  font-size: 0.9rem;
  white-space: pre-wrap;
  word-wrap: break-word;
}
</style>
