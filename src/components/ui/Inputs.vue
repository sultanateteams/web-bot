<template>
  <div class="mb-3">
    <label :for="id" class="form-label">{{ title }}</label>

    <b-form-input
      class="w-100"
      :id="id"
      v-model="modelValueProxy"
      :state="validationState"
      :type="supportedType"
    />

    <b-form-invalid-feedback v-if="required && !validationState">
      {{ errorMessage }}
    </b-form-invalid-feedback>
  </div>
</template>

<script lang="ts" setup>
import { computed, defineProps, defineEmits } from "vue";

const props = defineProps<{
  modelValue: string | number | null;
  id: string;
  title: string;
  required?: boolean;
  type: string;
}>();

const emit = defineEmits(["update:modelValue"]);

// Proxy orqali modelValue bilan bevosita ishlash
const modelValueProxy = computed({
  get: () => props.modelValue ?? "",
  set: (val: string | number) => emit("update:modelValue", val)
});

// Ruxsat etilgan input typelar
const allowedTypes = [
  "text",
  "password",
  "email",
  "number",
  "url",
  "tel",
  "search",
  "range",
  "date",
  "time",
  "color"
];

const supportedType = computed(() =>
  allowedTypes.includes(props.type) ? props.type : "text"
);

// Validatsiya
const validationState = computed(() => {
  if (!props.required) return null;
  const val = String(modelValueProxy.value).trim();

  if (supportedType.value === "number") {
    return !isNaN(Number(val));
  }

  return val.length >= 3 && val.length <= 100;
});

const errorMessage = computed(() => {
  if (supportedType.value === "number") {
    return "Faqat raqam bo‘lishi kerak.";
  }
  return "Matn 3 dan 100 belgigacha bo‘lishi kerak.";
});
</script>
