<template>
  <t-form
    ref="form"
    class="item-container login-password"
    :data="formData"
    :rules="FORM_RULES"
    label-width="0"
    @submit="onSubmit"
  >
    <t-form-item name="account">
      <t-input v-model="formData.account" size="large" placeholder="请输入用户名">
        <template #prefix-icon>
          <t-icon name="user" />
        </template>
      </t-input>
    </t-form-item>

    <t-form-item name="password">
      <t-input
        v-model="formData.password"
        size="large"
        :type="showPsw ? 'text' : 'password'"
        clearable
        placeholder="请输入密码"
      >
        <template #prefix-icon>
          <t-icon name="lock-on" />
        </template>
        <template #suffix-icon>
          <t-icon :name="showPsw ? 'browse' : 'browse-off'" @click="showPsw = !showPsw" />
        </template>
      </t-input>
    </t-form-item>

    <t-form-item class="btn-container">
      <t-button block size="large" type="submit" :loading="loading">管理员登录</t-button>
    </t-form-item>
  </t-form>
</template>
<script setup lang="ts">
import type { FormInstanceFunctions, FormRule, SubmitContext } from 'tdesign-vue-next';
import { MessagePlugin } from 'tdesign-vue-next';
import { ref } from 'vue';
import { useRoute, useRouter } from 'vue-router';

import type { LoginRequest } from '@/api/user';
import { login } from '@/api/user';
import { useUserStore } from '@/store';

const userStore = useUserStore();

const INITIAL_DATA = {
  account: '',
  password: '',
};

const FORM_RULES: Record<string, FormRule[]> = {
  account: [{ required: true, message: '请输入用户名', type: 'error' }],
  password: [{ required: true, message: '请输入密码', type: 'error' }],
};

const form = ref<FormInstanceFunctions>();
const formData = ref({ ...INITIAL_DATA });
const showPsw = ref(false);
const loading = ref(false);

const router = useRouter();
const route = useRoute();

const onSubmit = async (ctx: SubmitContext) => {
  if (ctx.validateResult === true) {
    loading.value = true;
    try {
      const loginData: LoginRequest = {
        login_type: 'password',
        username: formData.value.account,
        password: formData.value.password,
      };

      const result = await login(loginData);

      await userStore.setUserInfo(result);

      MessagePlugin.success('登录成功');
      const redirect = route.query.redirect as string;
      const redirectUrl = redirect ? decodeURIComponent(redirect) : '/dashboard';
      router.push(redirectUrl);
    } catch (error: any) {
      console.error('登录失败:', error);
      MessagePlugin.error(error.message || '登录失败，请重试');
    } finally {
      loading.value = false;
    }
  }
};
</script>
<style lang="less" scoped>
@import '../index.less';
</style>
