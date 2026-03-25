<script setup>
import { computed, ref, watch } from 'vue';
import { useDebounceFn } from '@vueuse/core';
import { useI18n } from 'vue-i18n';
import SearchAPI from 'dashboard/api/search';
import NextButton from 'dashboard/components-next/button/Button.vue';

const props = defineProps({
  show: { type: Boolean, default: false },
  selectedCount: { type: Number, default: 0 },
  isBusy: { type: Boolean, default: false },
});

const emit = defineEmits(['update:show', 'confirm', 'cancel']);

const { t } = useI18n();

const localShow = computed({
  get() {
    return props.show;
  },
  set(value) {
    emit('update:show', value);
    if (!value) emit('cancel');
  },
});

const query = ref('');
const isFetching = ref(false);
const conversations = ref([]);
const selectedConversationId = ref(null);

const selectedConversation = computed(() =>
  conversations.value.find(c => Number(c.id) === Number(selectedConversationId.value))
);

const fetchConversations = async q => {
  if (!q) {
    conversations.value = [];
    selectedConversationId.value = null;
    return;
  }
  isFetching.value = true;
  try {
    const { data } = await SearchAPI.conversations({ q, page: 1 });
    conversations.value = data?.payload?.conversations ?? [];
    if (
      selectedConversationId.value &&
      !conversations.value.some(
        c => Number(c.id) === Number(selectedConversationId.value)
      )
    ) {
      selectedConversationId.value = null;
    }
  } finally {
    isFetching.value = false;
  }
};

const debouncedFetch = useDebounceFn(fetchConversations, 250);

watch(query, q => {
  debouncedFetch(q?.trim());
});

watch(
  () => props.show,
  show => {
    if (!show) return;
    query.value = '';
    conversations.value = [];
    selectedConversationId.value = null;
  }
);

const onConfirm = () => {
  if (!selectedConversationId.value || props.isBusy) return;
  emit('confirm', { conversationId: selectedConversationId.value });
};
</script>

<template>
  <woot-modal v-model:show="localShow">
    <woot-modal-header
      :header-title="t('CONVERSATION.FORWARD.MODAL_TITLE')"
      :header-content="t('CONVERSATION.FORWARD.MODAL_SUBTITLE', { count: selectedCount })"
    />
    <div class="px-6 py-4">
      <div class="flex flex-col gap-3">
        <div class="flex flex-col gap-1">
          <label class="text-sm font-medium text-n-slate-12">
            {{ t('CONVERSATION.FORWARD.SEARCH_LABEL') }}
          </label>
          <input
            v-model="query"
            class="w-full rounded-md border border-n-strong bg-n-surface-1 px-3 py-2 text-sm text-n-slate-12 outline-none focus:ring-2 focus:ring-n-brand"
            type="text"
            :placeholder="t('CONVERSATION.FORWARD.SEARCH_PLACEHOLDER')"
          />
        </div>

        <div class="min-h-[10rem] rounded-md border border-n-strong bg-n-surface-1">
          <div v-if="isFetching" class="p-3 text-sm text-n-slate-11">
            {{ t('CONVERSATION.FORWARD.SEARCH_LOADING') }}
          </div>
          <div
            v-else-if="query && conversations.length === 0"
            class="p-3 text-sm text-n-slate-11"
          >
            {{ t('CONVERSATION.FORWARD.NO_RESULTS') }}
          </div>
          <ul v-else class="divide-y divide-n-strong">
            <li
              v-for="c in conversations"
              :key="c.id"
              class="flex items-start justify-between gap-3 p-3"
            >
              <label class="flex items-start gap-3 cursor-pointer w-full">
                <input
                  class="mt-1"
                  type="radio"
                  name="forwardConversation"
                  :value="c.id"
                  v-model="selectedConversationId"
                />
                <div class="min-w-0">
                  <div class="flex items-center gap-2 min-w-0">
                    <span class="text-sm font-medium text-n-slate-12 truncate">
                      {{ c?.contact?.name || t('CONVERSATION.FORWARD.UNKNOWN_CONTACT') }}
                    </span>
                    <span
                      v-if="c?.inbox?.name"
                      class="text-xs px-2 py-0.5 rounded-full bg-n-alpha-2 text-n-slate-11"
                    >
                      {{ c.inbox.name }}
                    </span>
                  </div>
                  <div class="text-xs text-n-slate-11 truncate">
                    #{{ c.id }}
                    <span v-if="c?.contact?.email"> · {{ c.contact.email }}</span>
                  </div>
                </div>
              </label>
            </li>
          </ul>
        </div>

        <div
          v-if="selectedConversation"
          class="rounded-md bg-n-alpha-2 px-3 py-2 text-xs text-n-slate-11"
        >
          {{ t('CONVERSATION.FORWARD.SELECTED_DESTINATION', { id: selectedConversation.id }) }}
        </div>
      </div>
    </div>
    <div class="flex items-center justify-end gap-2 px-6 py-4 border-t border-n-strong">
      <NextButton
        :label="t('CONVERSATION.FORWARD.CANCEL')"
        slate
        sm
        outline
        :disabled="isBusy"
        @click="localShow = false"
      />
      <NextButton
        :label="t('CONVERSATION.FORWARD.CONFIRM_FORWARD', { count: selectedCount })"
        icon="i-lucide-forward"
        primary
        sm
        :disabled="!selectedConversationId || isBusy"
        @click="onConfirm"
      />
    </div>
  </woot-modal>
</template>

