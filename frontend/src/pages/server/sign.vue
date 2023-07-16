<template>
  <v-container class="fill-height" fluid>
    <v-row justify="center">
      <v-col
        v-if="isEmpty(currentUser) && !loginAsOther && publicUsers.length > 0"
        sm="10"
        md="7"
        lg="5">
        <h1 class="text-h4 mb-6 text-center">{{ $t('login.selectUser') }}</h1>
        <v-row align="center" justify="center">
          <v-col
            v-for="publicUser in publicUsers"
            :key="publicUser.Id"
            cols="auto">
            <user-card :user="publicUser" @connect="setCurrentUser" />
          </v-col>
        </v-row>
        <v-row align="center" justify="center" dense class="mt-6">
          <v-col cols="11" sm="6" class="d-flex justify-center">
            <v-btn
              block
              size="large"
              variant="elevated"
              @click="loginAsOther = true">
              {{ $t('login.manualLogin') }}
            </v-btn>
          </v-col>
          <v-col cols="11" sm="6" class="d-flex justify-center">
            <v-btn block to="/server/select" size="large" variant="elevated">
              {{ $t('login.changeServer') }}
            </v-btn>
          </v-col>
        </v-row>
      </v-col>
      <v-col
        v-else-if="
          !isEmpty(currentUser) ||
          loginAsOther ||
          (publicUsers.length === 0 && $remote.auth.currentServer?.ServerName)
        "
        sm="6"
        md="6"
        lg="5">
        <h1 v-if="!isEmpty(currentUser)" class="text-h4 mb-3 text-center">
          {{ $t('login.loginAs', { name: currentUser.Name }) }}
        </h1>
        <h1 v-else class="text-h4 text-center">
          Sign up
        </h1>
        <h5 class="text-center mb-3 text--disabled">
          {{ $remote.auth.currentServer?.ServerName }}
        </h5>

        <v-form v-model="valid" :disabled="loading" @submit.prevent="userSignUp">
          <v-text-field
            v-if="isEmpty(user)"
            v-model="login.username"
            variant="outlined"
            hide-details
            autofocus
            :label="$t('username')"
            :rules="rules" />
          <v-text-field
            v-model="login.password"
            variant="outlined"
            hide-details
            class="mt-4"
            :label="$t('password')"
            :append-inner-icon="showPassword ? IconEyeOff : IconEye"
            :type="showPassword ? 'text' : 'password'"
            @click:append="() => (showPassword = !showPassword)" />
          <v-row align="center" no-gutters class="mt-6 mb-6">
            <v-col class="ml-2">
              <v-btn
                :disabled="!valid"
                :loading="loading"
                block
                size="large"
                color="primary"
                variant="elevated"
                type="submit">
                Sign up
              </v-btn>
            </v-col>
            <v-col>
              <v-btn
                block
                size="large"
                variant="elevated"
                @click="navigateToSignIn()">
                {{ $t('signIn') }}
              </v-btn>
            </v-col>
          </v-row>
        </v-form>
        <p class="text-p mt-6 text-center">{{ disclaimer }}</p>
      </v-col>
    </v-row>
  </v-container>
</template>

<route lang="yaml">
meta:
  layout: server
</route>

<script setup lang="ts">
import { ref } from 'vue';
import { isEmpty } from 'lodash-es';
import { UserDto } from '@jellyfin/sdk/lib/generated-client';
import { getBrandingApi } from '@jellyfin/sdk/lib/utils/api/branding-api';
import { getUserApi } from '@jellyfin/sdk/lib/utils/api/user-api';
import { getSystemApi } from '@jellyfin/sdk/lib/utils/api/system-api';
import { useRoute, useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
import { userLibrariesStore } from '@/store';
import { useRemote } from '@/composables';

const { t } = useI18n();
const route = useRoute();
const router = useRouter();
const remote = useRemote();
const login = ref({ username: '', password: '', rememberMe: true });
const loading = ref(false);
const api = remote.sdk.oneTimeSetup(
  remote.auth.currentServer?.PublicAddress || ''
);
const userLibraries = userLibrariesStore();

route.meta.title = t('login.login');

try {
  await getSystemApi(api).getPublicSystemInfo();
} catch {
  router.replace('/server/select');
}

const brandingData = (await getBrandingApi(api).getBrandingOptions()).data;
const publicUsers = (await getUserApi(api).getPublicUsers({})).data;

const disclaimer = brandingData.LoginDisclaimer;

const loginAsOther = ref(false);
const currentUser = ref<UserDto>({});

/**
 * Sets the current user for public user login
 */
async function setCurrentUser(user: UserDto): Promise<void> {
  if (!user.HasPassword && user.Name) {
    // If the user doesn't have a password, avoid showing the password form
    await remote.auth.loginUser(user.Name, '');
    router.replace('/');
  } else {
    currentUser.value = user;
  }
}

/**
 * Resets the currently selected user
 */
function resetCurrentUser(): void {
  currentUser.value = {};
  loginAsOther.value = false;
}

/**
 * navigateToSignUp page
 */
function navigateToSignIn(): void {
  router.push('/server/login');
}

/**
 * navigateToSignUp page
 */
async function userSignUp(): Promise<void> {
  loading.value = true;

  console.log(login.value);

  try {
    await remote.auth.createUser(login.value.username, login.value.password);

    await userLibraries.refresh();

    router.replace('/');
  } finally {
    loading.value = false;
  }
}
</script>
