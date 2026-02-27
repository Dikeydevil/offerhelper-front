<script setup>
const props = defineProps({
  email: String,
  password: String,
  isAuthed: Boolean,
  me: Object,
  authError: String,
});
const emit = defineEmits(['update:email', 'update:password', 'login', 'register', 'logout']);
</script>

<template>
  <section class="auth card">
    <h2>Вход</h2>
    <p v-if="me">Вы вошли как {{ me.email }}</p>
    <p v-if="authError" class="error">{{ authError }}</p>

    <div v-if="!isAuthed">
      <input
        :value="email"
        @input="emit('update:email', $event.target.value)"
        type="email"
        placeholder="Email"
        class="auth-input"
      />
      <input
        :value="password"
        @input="emit('update:password', $event.target.value)"
        type="password"
        placeholder="Пароль"
        class="auth-input"
      />
      <div class="auth-actions">
        <button class="btn primary" @click="emit('login')">Войти</button>
        <button class="btn ghost" @click="emit('register')">Регистрация</button>
      </div>
    </div>

    <div v-else>
      <button class="btn ghost" @click="emit('logout')">Выйти</button>
    </div>
  </section>
</template>

<style scoped>
.auth {
  margin-bottom: 16px;
  background: #ffffff;
  border-radius: 14px;
  padding: 16px 18px 18px;
  box-shadow: 0 10px 26px rgba(15, 23, 42, 0.06);
  border: 1px solid #e5e7eb;
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
.btn.ghost {
  background: transparent;
  border-color: #d1d5db;
  color: #374151;
}
.error {
  color: #b91c1c;
}
</style>
