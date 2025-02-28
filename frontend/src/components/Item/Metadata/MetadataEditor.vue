<template>
  <v-card
    v-if="metadata"
    height="100%"
    class="d-flex flex-column metadata-editor">
    <v-card-title>{{ t('editMetadata') }}</v-card-title>
    <v-card-subtitle class="pb-3">
      {{ metadata.Path }}
    </v-card-subtitle>

    <v-divider />

    <v-card-text
      class="pa-0 flex-grow-1"
      :class="{
        'd-flex': !$vuetify.display.mobile,
        'flex-row': !$vuetify.display.mobile
      }">
      <v-tabs
        v-model="tabName"
        :direction="$vuetify.display.mobile ? 'horizontal' : 'vertical'">
        <v-tab value="general">{{ t('general') }}</v-tab>
        <v-tab value="details">{{ t('details') }}</v-tab>
        <v-tab value="castAndCrew">{{ t('castAndCrew') }}</v-tab>
        <v-tab value="images">{{ t('images') }}</v-tab>
      </v-tabs>
      <v-window v-model="tabName" class="pa-2 flex-fill">
        <v-window-item value="general">
          <v-text-field
            v-model="metadata.Name"
            variant="outlined"
            :label="t('metadata.title')" />
          <v-text-field
            v-model="metadata.OriginalTitle"
            variant="outlined"
            :label="t('originalTitle')" />
          <v-text-field
            v-model="metadata.ForcedSortName"
            variant="outlined"
            :label="t('sortTitle')" />
          <v-text-field
            v-model="tagLine"
            variant="outlined"
            :label="t('tagline')" />
          <v-textarea
            v-model="metadata.Overview"
            variant="outlined"
            no-resize
            rows="4"
            :label="t('overview')" />
        </v-window-item>
        <v-window-item value="details">
          <date-input
            :value="dateCreated"
            :label="t('dateAdded')"
            @update:date="
              (value) => formatAndAssignDate('DateCreated', value)
            " />
          <v-row>
            <v-col sm="6" cols="12">
              <v-text-field
                v-model="metadata.CommunityRating"
                variant="outlined"
                :label="t('communityRating')" />
            </v-col>
            <v-col sm="6" cols="12">
              <v-text-field
                v-model="metadata.CriticRating"
                variant="outlined"
                :label="t('criticRating')" />
            </v-col>
          </v-row>

          <date-input
            :value="premiereDate"
            :label="t('releaseDate')"
            @update:date="
              (value) => formatAndAssignDate('PremiereDate', value)
            " />
          <v-text-field
            v-model="metadata.ProductionYear"
            variant="outlined"
            :label="t('year')" />
          <v-text-field
            v-model="metadata.OfficialRating"
            variant="outlined"
            :label="t('parentalRating')" />
          <v-text-field
            v-model="metadata.CustomRating"
            variant="outlined"
            :label="t('customRating')" />
          <v-combobox
            v-model="genresModel"
            :items="genres"
            :label="t('genres')"
            hide-selected
            multiple
            variant="outlined"
            :hide-no-data="false"
            chips
            closable-chips />
          <v-combobox
            v-model="tagsModel"
            :label="t('tags')"
            multiple
            variant="outlined"
            chips
            closable-chips />
        </v-window-item>
        <v-window-item value="castAndCrew">
          <v-list lines="two">
            <v-list-item :title="t('addNewPerson')" @click="onPersonAdd">
              <template #append>
                <v-avatar>
                  <v-icon>
                    <i-mdi-plus-circle />
                  </v-icon>
                </v-avatar>
              </template>
            </v-list-item>
            <v-list-item
              v-for="(item, i) in metadata.People"
              :key="`${item.Id}-${i}`"
              :title="item.Name ?? undefined"
              :subtitle="(item.Role || item.Type) ?? undefined"
              @click="onPersonEdit(item)">
              <template #prepend>
                <v-avatar>
                  <v-img
                    v-if="item.Id && item.PrimaryImageTag"
                    :src="
                      remote.sdk.api?.getItemImageUrl(
                        item.Id,
                        ImageType.Primary
                      )
                    " />
                  <v-icon v-else class="bg-grey-darken-3">
                    <i-mdi-account />
                  </v-icon>
                </v-avatar>
              </template>
              <template #append>
                <v-avatar @click.stop="onPersonDel(i)">
                  <v-icon>
                    <i-mdi-delete />
                  </v-icon>
                </v-avatar>
              </template>
            </v-list-item>
          </v-list>
          <person-editor
            :person="person"
            @update:person="onPersonSave"
            @close="person = undefined" />
        </v-window-item>
        <v-window-item value="images">
          <image-editor :metadata="metadata" />
        </v-window-item>
      </v-window>
    </v-card-text>

    <v-divider />
    <v-card-actions
      class="d-flex align-center pa-3"
      :class="{
        'justify-end': !$vuetify.display.mobile,
        'justify-center': $vuetify.display.mobile
      }">
      <v-btn
        variant="flat"
        width="8em"
        color="secondary"
        class="mr-1"
        @click="emit('cancel')">
        {{ t('cancel') }}
      </v-btn>
      <v-btn
        variant="flat"
        width="8em"
        color="primary"
        :loading="loading"
        @click="saveMetadata">
        {{ t('save') }}
      </v-btn>
    </v-card-actions>
  </v-card>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import { pick, set } from 'lodash-es';
import { AxiosError } from 'axios';
import {
  BaseItemDto,
  BaseItemPerson,
  ImageType
} from '@jellyfin/sdk/lib/generated-client';
import { getLibraryApi } from '@jellyfin/sdk/lib/utils/api/library-api';
import { getUserLibraryApi } from '@jellyfin/sdk/lib/utils/api/user-library-api';
import { getGenresApi } from '@jellyfin/sdk/lib/utils/api/genres-api';
import { getItemUpdateApi } from '@jellyfin/sdk/lib/utils/api/item-update-api';
import { format, formatISO } from 'date-fns';
import { useDateFns, useRemote, useSnackbar } from '@/composables';

const props = defineProps<{ itemId: string }>();

const emit = defineEmits<{
  save: [];
  'update:forceRefresh': [];
  cancel: [];
}>();

const { t } = useI18n();
const remote = useRemote();

const metadata = ref<BaseItemDto>();
const menu = ref(false);
const person = ref<BaseItemPerson>();
const genres = ref<string[]>([]);
const loading = ref(false);
const tabName = ref<string>();
const genresModel = computed({
  get() {
    return metadata.value?.Genres === null ? undefined : metadata.value?.Genres;
  },
  set(newVal) {
    if (Array.isArray(newVal) && metadata.value) {
      metadata.value.Genres = newVal;
    }
  }
});
const tagsModel = computed({
  get() {
    return metadata.value?.Tags === null ? undefined : metadata.value?.Tags;
  },
  set(newVal) {
    if (Array.isArray(newVal) && metadata.value) {
      metadata.value.Tags = newVal;
    }
  }
});

const premiereDate = computed(() => {
  if (!metadata.value?.PremiereDate) {
    return '';
  }

  return useDateFns(format, new Date(metadata.value.PremiereDate), 'yyyy-MM-dd')
    .value;
});

const dateCreated = computed(() => {
  if (!metadata.value?.DateCreated) {
    return '';
  }

  return useDateFns(format, new Date(metadata.value.DateCreated), 'yyyy-MM-dd')
    .value;
});

const tagLine = computed({
  get: () => metadata.value?.Taglines?.[0] ?? '',
  set: (v) => {
    if (metadata.value) {
      if (!metadata.value?.Taglines) {
        metadata.value.Taglines = [];
      }

      metadata.value.Taglines[0] = v;
    }
  }
});

/**
 * Fetch data ancestors for the current item
 */
async function getData(): Promise<void> {
  const itemInfo = (
    await remote.sdk.newUserApi(getUserLibraryApi).getItem({
      userId: remote.auth.currentUserId ?? '',
      itemId: props.itemId
    })
  ).data;

  metadata.value = itemInfo;

  if (!metadata.value?.Id) {
    return;
  }

  const ancestors = await remote.sdk.newUserApi(getLibraryApi).getAncestors({
    userId: remote.auth.currentUserId ?? '',
    itemId: metadata.value.Id
  });
  const libraryInfo = ancestors.data.find(
    (index) => index.Type === 'CollectionFolder'
  );

  if (!libraryInfo?.Id) {
    return;
  }

  await getGenres(libraryInfo.Id);
}

/**
 * Get genres associated with the current item
 */
async function getGenres(parentId: string): Promise<void> {
  genres.value =
    (
      await remote.sdk.newUserApi(getGenresApi).getGenres({
        parentId
      })
    ).data.Items?.map((index) => index.Name).filter(
      (genre): genre is string => !!genre
    ) ?? [];
}

/**
 * Save metadata for the current item
 */
async function saveMetadata(): Promise<void> {
  if (!metadata.value || !metadata.value.Id) {
    return;
  }

  const item = pick(metadata.value, [
    'Id',
    'Name',
    'OriginalTitle',
    'ForcedSortName',
    'CommunityRating',
    'CriticRating',
    'IndexNumber',
    'AirsBeforeSeasonNumber',
    'AirsAfterSeasonNumber',
    'AirsBeforeEpisodeNumber',
    'ParentIndexNumber',
    'DisplayOrder',
    'Album',
    'AlbumArtists',
    'ArtistItems',
    'Overview',
    'Status',
    'AirDays',
    'AirTime',
    'Genres',
    'Tags',
    'Studios',
    'PremiereDate',
    'DateCreated',
    'EndDate',
    'ProductionYear',
    'AspectRatio',
    'Video3DFormat',
    'OfficialRating',
    'CustomRating',
    'People',
    'LockData',
    'LockedFields',
    'ProviderIds',
    'PreferredMetadataLanguage',
    'PreferredMetadataCountryCode',
    'Taglines'
  ]);

  try {
    loading.value = true;

    if (!metadata.value.Id) {
      throw new Error('Expected metadata to have id');
    }

    await remote.sdk.newUserApi(getItemUpdateApi).updateItem({
      itemId: metadata.value?.Id,
      baseItemDto: item
    });
    emit('save');
    useSnackbar(t('saved'), 'success');
  } catch (error) {
    // TODO: This whole block should be removed - we should verify that the data is correct client-side before posting to server
    // not expecting bad request messages.
    // TODO: Revise similar blocks like this through the entire codebase.
    let errorMessage = t('unexpectedError');

    if (error instanceof AxiosError && error.response?.status === 400) {
      errorMessage = t('badRequest');
    }

    useSnackbar(errorMessage, 'error');
  } finally {
    loading.value = false;
  }
}

/**
 * Formats and updates dates
 */
function formatAndAssignDate(key: keyof BaseItemDto, date: string): void {
  if (!metadata.value) {
    return;
  }

  menu.value = false;
  set(metadata.value, key, formatISO(new Date(date)));
}

/**
 * Handle adding a person
 */
function onPersonAdd(): void {
  onPersonEdit({});
}

/**
 * Handle editing a person
 */
function onPersonEdit(item: BaseItemPerson): void {
  person.value = item;
}

/**
 * Handle saving a person after editing
 */
function onPersonSave(item: BaseItemPerson): void {
  if (!metadata.value?.People) {
    return;
  }

  if (item.Id) {
    metadata.value.People = metadata.value.People.map((p) =>
      p.Id === item.Id ? item : p
    );
  } else {
    // undefined id means that the person was newly added
    metadata.value.People.push(item);
  }

  person.value = undefined;
}

/**
 * Handle deleting a person
 */
function onPersonDel(index: number): void {
  if (!metadata.value?.People) {
    return;
  }

  metadata.value.People.splice(index, 1);
}

watch(() => props.itemId, getData, { immediate: true });
</script>
