<template>
  <div class="login-wrapper">
    <login-header />

    <div class="login-container">
      <div class="title-container">
        <h1 class="title margin-no">Claude Code Relay</h1>
      </div>

      <t-card :class="['main-card', { 'main-card--expanded': expanded }]">
        <t-tabs v-model="tab">
          <t-tab-panel value="apikey" label="用量查询">
            <api-key-query @has-data="onDataChange" @clear-data="onClear" />
          </t-tab-panel>
          <t-tab-panel value="install" label="安装教程">
            <help-page />
          </t-tab-panel>
          <t-tab-panel value="login" label="管理登录">
            <login />
          </t-tab-panel>
        </t-tabs>
      </t-card>
      <tdesign-setting />
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref, watch } from 'vue';

import TdesignSetting from '@/layouts/setting.vue';
import HelpPage from '@/pages/help/index.vue';

import ApiKeyQuery from './components/ApiKeyQuery.vue';
import LoginHeader from './components/Header.vue';
import Login from './components/Login.vue';

defineOptions({ name: 'LoginIndex' });

const tab = ref('apikey');
const expanded = ref(false);

watch(tab, (newTab) => {
  expanded.value = newTab === 'install';
});

const onDataChange = (hasData: boolean) => {
  if (tab.value === 'apikey') {
    expanded.value = hasData;
  }
};

const onClear = () => {
  expanded.value = false;
};
</script>
<style lang="less" scoped>
@import './index.less';
</style>
