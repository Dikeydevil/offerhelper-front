<script setup>
const props = defineProps({
  me: Object,
  isAdmin: Boolean,
  changePasswordOld: String,
  changePasswordNew: String,
  changePasswordError: String,
  changePasswordSuccess: String,
  users: { type: Array, default: () => [] },
  usersError: String,
  usersLoading: Boolean,
  usersPage: Number,
  usersPageSize: Number,
});
const emit = defineEmits([
  'update:change-password-old',
  'update:change-password-new',
  'change-password',
  'load-users',
  'set-page',
  'reset-password',
]);
</script>

<template>
  <section class="card cabinet" v-if="me">
    <h2>Кабинет</h2>

    <div class="cabinet-profile">
      <div><strong>Email:</strong> {{ me.email }}</div>
      <div><strong>Роль:</strong> {{ me.role }}</div>
      <div><strong>ID:</strong> {{ me.id }}</div>
      <div><strong>Создан:</strong> {{ new Date(me.created_at).toLocaleString() }}</div>
    </div>

    <div class="cabinet-change-password">
      <h3>Смена пароля</h3>
      <p v-if="changePasswordError" class="error">{{ changePasswordError }}</p>
      <p v-if="changePasswordSuccess" class="success">{{ changePasswordSuccess }}</p>
      <input
        :value="changePasswordOld"
        @input="emit('update:change-password-old', $event.target.value)"
        type="password"
        placeholder="Старый пароль"
        class="auth-input"
      />
      <input
        :value="changePasswordNew"
        @input="emit('update:change-password-new', $event.target.value)"
        type="password"
        placeholder="Новый пароль"
        class="auth-input"
      />
      <button class="btn primary" type="button" @click="emit('change-password')">
        Изменить пароль
      </button>
    </div>

    <div v-if="isAdmin" class="cabinet-admin">
      <div class="cabinet-admin-header">
        <h3>Управление пользователями</h3>
        <button class="btn ghost" type="button" @click="emit('load-users')">
          Обновить список
        </button>
      </div>
      <p v-if="usersError" class="error">{{ usersError }}</p>
      <p v-if="usersLoading">Загружаем пользователей…</p>

      <div v-if="users.length" class="users-table-wrapper">
        <table class="users-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Email</th>
              <th>Роль</th>
              <th>Создан</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="u in users" :key="u.id">
              <td>{{ u.id }}</td>
              <td>{{ u.email }}</td>
              <td>{{ u.role }}</td>
              <td>{{ new Date(u.created_at).toLocaleString() }}</td>
              <td>
                <button
                  class="btn ghost btn-xs"
                  type="button"
                  @click="emit('reset-password', u.id)"
                >
                  Сбросить пароль
                </button>
              </td>
            </tr>
          </tbody>
        </table>

        <div class="users-pagination">
          <button
            class="btn ghost btn-xs"
            type="button"
            :disabled="usersPage === 1"
            @click="emit('set-page', usersPage - 1)"
          >
            ← Назад
          </button>
          <span>Страница {{ usersPage }}</span>
          <button
            class="btn ghost btn-xs"
            type="button"
            :disabled="!users.length || users.length < usersPageSize"
            @click="emit('set-page', usersPage + 1)"
          >
            Вперёд →
          </button>
        </div>
      </div>

      <p v-else-if="!usersLoading && !usersError">
        Пользователей пока нет.
      </p>
    </div>
  </section>
</template>

<style scoped>
.cabinet {
  margin-top: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.cabinet-profile {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 8px;
  font-size: 0.9rem;
}
.cabinet-change-password h3,
.cabinet-admin h3 {
  margin-bottom: 6px;
}
.cabinet-change-password {
  max-width: 420px;
}
.success {
  color: #16a34a;
}
.error {
  color: #b91c1c;
}
.cabinet-admin {
  margin-top: 4px;
}
.cabinet-admin-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}
.users-table-wrapper {
  border-radius: 10px;
  border: 1px solid #e5e7eb;
  overflow: hidden;
  background: #f9fafb;
}
.users-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9rem;
}
.users-table th,
.users-table td {
  padding: 6px 8px;
  border-bottom: 1px solid #e5e7eb;
}
.users-table th {
  background: #eef2ff;
  text-align: left;
  font-weight: 600;
}
.users-table tr:nth-child(even) td {
  background: #f9fafb;
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
.btn-xs {
  padding: 4px 10px;
  font-size: 0.8rem;
}
.users-pagination {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 8px;
  padding: 6px 8px;
  font-size: 0.85rem;
}
.auth-input {
  display: block;
  width: 100%;
  margin-bottom: 8px;
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid #d1d5db;
}
</style>
