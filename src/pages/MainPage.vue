<script setup>
import ResumeLayout from '../components/ResumeLayout.vue';

const props = defineProps({
  apiBase: String,
  accessToken: [String, Object],
});
const emit = defineEmits(['request-done']);
</script>

<template>
  <ResumeLayout
    :api-base="apiBase"
    :access-token="accessToken"
    @request-done="emit('request-done')"
  />
</template>
