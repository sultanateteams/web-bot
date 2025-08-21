<template>
  <div class="p-4">
    <b-form @submit.stop.prevent="onSubmit">
      <h1>{{ isEdit ? "Ma'lumotlarni tahrirlash" : "Ro'yxatdan o'tish" }}</h1>

      <Inputs
        type="text"
        id="first-name"
        title="Ism: (*majburiy)"
        v-model="user.firstName"
        v-if="requiredFields.includes('firstName')"
        :required="true"
      />

      <Inputs
        type="text"
        id="last-name"
        title="Sharifingiz: (*majburiy)"
        v-model="user.lastName"
        v-if="requiredFields.includes('lastName')"
        :required="true"
      />

      <b-form-group
        label="Viloyatingizni tanlang: (*majburiy)"
        label-for="viloyat"
        v-if="requiredFields.includes('viloyat')"
      >
        <b-form-select
          id="viloyat"
          v-model="user.viloyat"
          :options="region"
          required
        />
      </b-form-group>

      <b-form-group
        label="Tumaningizni tanlang: (*majburiy)"
        label-for="tuman"
        v-if="requiredFields.includes('tuman')"
      >
        <b-form-select
          id="tuman"
          v-model="user.tuman"
          :options="district"
          :disabled="!district.length"
          required
        />
      </b-form-group>

      <Inputs
        type="text"
        id="full_address"
        title="Yashash manzilingiz (mahalla, ko'cha, uy, raqam)"
        v-model="user.fullAddress"
        v-if="requiredFields.includes('fullAddress')"
      />

      <Inputs
        type="date"
        id="birth-date"
        title="Tug'ilgan kuningiz"
        v-model="user.birthDate"
        v-if="requiredFields.includes('birthDate')"
      />

      <Inputs
        type="number"
        id="postcode"
        title="Po'chta indeksingiz"
        v-model="user.postcode"
        v-if="requiredFields.includes('postcode')"
      />

      <MaskNumber
        id="phone_number"
        title="Telefon raqamingiz"
        v-model="user.phone_number"
        v-if="requiredFields.includes('phone_number')"
      />
    </b-form>
  </div>
</template>

<script lang="ts" setup>
import { onMounted, reactive, watch, ref, computed } from "vue";

import Inputs from "./ui/Inputs.vue";
import MaskNumber from "./ui/MaskNumber.vue";
import { viloyat, tuman } from "../constants/regions";

const region = viloyat
  .map((el) => ({ ...el, value: el.number, text: el.name1 }))
  .sort((a, b) => a.number - b.number);

const district = ref([]);
const Telegram = window.Telegram.WebApp;

// --- Query paramlarni olish ---
const queryParams = new URLSearchParams(window.location.search);
const requiredRaw = queryParams.get("required");
const currentUserRaw = queryParams.get("currentUser");

const requiredFields = requiredRaw
  ? requiredRaw.split("|")
  : ["firstName", "lastName", "viloyat", "tuman", "phone_number"];

// --- Formadagi foydalanuvchi ma’lumotlari ---
const user = reactive({
  firstName: "",
  lastName: "",
  viloyat: null,
  viloyatInfo: {},
  tuman: null,
  tumanInfo: {},
  fullAddress: "",
  birthDate: "",
  postcode: "",
  phone_number: "",
});

// Agar currentUser bo‘lsa — formani to‘ldirish
if (currentUserRaw) {
  try {
    const parsed = JSON.parse(decodeURIComponent(currentUserRaw));
    Object.assign(user, {
      firstName: parsed.first_name ?? "",
      lastName: parsed.last_name ?? "",
      phone_number: parsed.phone_number ?? "",
      tuman: parsed.tuman ?? null,
      viloyat: parsed.viloyat ?? null,
      fullAddress: parsed.full_address ?? "",
      birthDate: parsed.birth_date ?? "",
      postcode: parsed.postcode ?? "",
    });
  } catch (err) {
    console.error("❌ currentUser parse error:", err);
  }
}

// Edit yoki yangi ro‘yxatdan o‘tish
const isEdit = computed(() => !!currentUserRaw);

// --- Tumanlarni viloyat bo‘yicha chiqarish ---
const districtON = () => {
  district.value = tuman
    .filter((el) => el.parent_number == user.viloyat)
    .map((el) => ({ value: el.number, text: el.name1 }))
    .sort((a, b) => a.number - b.number);
};

// --- Lifecycle ---
onMounted(() => {
  Telegram.ready();
  districtON();

  Telegram.onEvent("mainButtonClicked", () => {
    const queryId = Telegram.initDataUnsafe?.query_id;

    user.viloyatInfo = viloyat.find((el) => user.viloyat == el.number);
    user.tumanInfo = tuman.find((el) => user.tuman == el.number);

    // Backendga snake_case formatida yuborish
    const payload = JSON.stringify({
      user: {
        first_name: user.firstName,
        last_name: user.lastName,
        viloyat: user.viloyat,
        viloyat_info: user.viloyatInfo,
        tuman: user.tuman,
        tuman_info: user.tumanInfo,
        full_address: user.fullAddress,
        birth_date: user.birthDate,
        postcode: user.postcode,
        phone_number: user.phone_number,
      },
      queryId,
    });

    if (queryId) {
      fetch("https://telegram-bota-da4625226d63.herokuapp.com/web-data", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: payload,
      });
    } else {
      Telegram.sendData(payload);
    }
  });
});

// --- Viloyat tanlanganda tumanlarni yangilash ---
watch(
  () => user.viloyat,
  () => {
    districtON();
  },
  { immediate: true }
);

// --- Forma to‘liq bo‘lganda MainButton chiqishi ---
watch(
  () => [user.firstName, user.lastName, user.viloyat, user.tuman],
  ([firstName, lastName, viloyat, tuman]) => {
    if (firstName?.trim() && lastName?.trim() && viloyat && tuman) {
      Telegram.MainButton.text = isEdit.value ? "Update" : "Register";
      Telegram.MainButton.show();
    } else {
      Telegram.MainButton.hide();
    }
  },
  { immediate: true }
);

const onSubmit = () => {
  districtON();
};
</script>
