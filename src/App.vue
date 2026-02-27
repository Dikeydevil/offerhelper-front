<script setup>
import { ref, computed, onMounted } from 'vue';
import AuthSection from './components/AuthSection.vue';
import HistorySidebar from './components/HistorySidebar.vue';
import MainPage from './pages/MainPage.vue';
import CabinetPage from './pages/CabinetPage.vue';

const API_BASE = 'http://127.0.0.1:8000';

// auth
const email = ref('');
const password = ref('');
const accessToken = ref(null);
const authError = ref('');
const me = ref(null);

const isAuthed = computed(() => !!accessToken.value);
const isAdmin = computed(() => me.value && me.value.role === 'admin');

// routing
const currentPage = ref('main'); // main | cabinet
function goTo(page) {
  currentPage.value = page;
}

// history
const requests = ref([]);
const requestsError = ref('');

// profile / cabinet
const changePasswordOld = ref('');
const changePasswordNew = ref('');
const changePasswordError = ref('');
const changePasswordSuccess = ref('');

const users = ref([]);
const usersError = ref('');
const usersLoading = ref(false);
const usersPage = ref(1);
const usersPageSize = ref(20);

// ---------- API helpers ----------
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
    me.value = await res.json(); // { id, email, role, created_at }
  } catch {
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

// auth endpoints
async function register() {
  authError.value = '';
  try {
    const res = await fetch(`${API_BASE}/auth/register`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email: email.value, password: password.value }),
    });
    if (!res.ok) throw new Error((await res.text()) || 'Ошибка регистрации');
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
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body,
    });
    if (!res.ok) throw new Error((await res.text()) || 'Ошибка входа');

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
  requests.value = [];
}

// history
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

// change password
async function changePassword() {
  changePasswordError.value = '';
  changePasswordSuccess.value = '';
  if (!changePasswordOld.value || !changePasswordNew.value) {
    changePasswordError.value = 'Заполните оба поля.';
    return;
  }
  try {
    const res = await fetch(`${API_BASE}/auth/change-password`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${accessToken.value}`,
      },
      body: JSON.stringify({
        old_password: changePasswordOld.value,
        new_password: changePasswordNew.value,
      }),
    });
    if (!res.ok) throw new Error((await res.text()) || 'Не удалось сменить пароль');
    const data = await res.json();
    if (data.status === 'ok') {
      changePasswordSuccess.value = 'Пароль успешно изменён.';
      changePasswordOld.value = '';
      changePasswordNew.value = '';
    } else {
      changePasswordError.value = 'Неизвестный ответ сервера.';
    }
  } catch (e) {
    changePasswordError.value = e.message || 'Ошибка при смене пароля.';
  }
}

// admin users
async function loadUsers() {
  if (!isAdmin.value) return;
  usersError.value = '';
  usersLoading.value = true;
  try {
    const params = new URLSearchParams({
      page: String(usersPage.value),
      page_size: String(usersPageSize.value),
    });
    const res = await fetch(`${API_BASE}/admin/users?` + params.toString(), {
      headers: { Authorization: `Bearer ${accessToken.value}` },
    });
    if (!res.ok) throw new Error((await res.text()) || 'Не удалось загрузить пользователей');
    users.value = await res.json();
  } catch (e) {
    usersError.value = e.message || 'Ошибка загрузки пользователей.';
  } finally {
    usersLoading.value = false;
  }
}

async function resetUserPassword(userId) {
  if (!isAdmin.value) return;
  try {
    const res = await fetch(`${API_BASE}/admin/users/${userId}/reset-password`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${accessToken.value}` },
    });
    if (!res.ok) throw new Error((await res.text()) || 'Не удалось сбросить пароль');
    const data = await res.json();
    window.prompt(
      `Новый пароль для пользователя #${data.user_id}. Скопируйте его:`,
      data.new_password
    );
  } catch (e) {
    alert(e.message || 'Ошибка при сбросе пароля.');
  }
}
</script>

<template>
  <div class="page">
    <header class="site-header">
      <div class="site-header__inner">
        <div class="brand">
          <div class="brand-mark">oh</div>
          <span class="brand-name">offerhelper</span>
        </div>
        <nav class="nav">
          <button class="nav-link nav-button" type="button" @click="goTo('main')">
            Главная
          </button>
          <button
            class="nav-link nav-button"
            type="button"
            @click="goTo('cabinet')"
            :disabled="!isAuthed"
          >
            Кабинет
          </button>
        </nav>
      </div>
    </header>

    <AuthSection
      :email="email"
      :password="password"
      :is-authed="isAuthed"
      :me="me"
      :auth-error="authError"
      @update:email="email = $event"
      @update:password="password = $event"
      @login="login"
      @register="register"
      @logout="logout"
    />

    <div v-if="currentPage === 'main'" class="main-layout-with-history">
      <HistorySidebar
        v-if="isAuthed"
        :requests="requests"
        :error="requestsError"
        @refresh="loadRequests"
      />
      <MainPage
        :api-base="API_BASE"
        :access-token="accessToken"
        @request-done="loadRequests"
      />
    </div>

    <CabinetPage
      v-else-if="currentPage === 'cabinet'"
      :me="me"
      :is-admin="isAdmin"
      :change-password-old="changePasswordOld"
      :change-password-new="changePasswordNew"
      :change-password-error="changePasswordError"
      :change-password-success="changePasswordSuccess"
      :users="users"
      :users-error="usersError"
      :users-loading="usersLoading"
      :users-page="usersPage"
      :users-page-size="usersPageSize"
      @update:change-password-old="changePasswordOld = $event"
      @update:change-password-new="changePasswordNew = $event"
      @change-password="changePassword"
      @load-users="loadUsers"
      @set-page="(p) => { usersPage = p; loadUsers(); }"
      @reset-password="resetUserPassword"
    />

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
    sans-se
