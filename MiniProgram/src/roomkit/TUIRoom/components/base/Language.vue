<template>
  <svg-icon
    class="language-icon" icon-name="lang" size="large" @click="handleChange"
  ></svg-icon>
</template>

<script setup lang="ts">
import SvgIcon from '../common/SvgIcon.vue';
import { useBasicStore } from '../../stores/basic';
import i18n from '../../locales/index';

const basicStore = useBasicStore();

const handleChange = (): void => {
  switch (i18n.global.locale.value) {
    case 'en-US':
      i18n.global.locale.value = 'zh-CN';
      basicStore.setLang('zh-CN');
      uni.setStorageSync('tuiRoom-language', 'zh-CN');
      break;
    case 'zh-CN':
      i18n.global.locale.value = 'en-US';
      basicStore.setLang('en-US');
      uni.setStorageSync('tuiRoom-language', 'en-US');
      break;
  }
};

</script>

<style>
.language {
  cursor: pointer;
}
</style>
