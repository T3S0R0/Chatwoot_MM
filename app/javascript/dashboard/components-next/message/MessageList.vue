<script setup>
import { defineProps, computed, reactive, provide, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import Message from './Message.vue';
import { MESSAGE_TYPES } from './constants.js';
import { useCamelCase } from 'dashboard/composables/useTransformKeys';
import { useMapGetter } from 'dashboard/composables/store.js';
import MessageApi from 'dashboard/api/inbox/message.js';
import NextButton from 'dashboard/components-next/button/Button.vue';
import ForwardMessagesModal from './ForwardMessagesModal.vue';
import { useAlert } from 'dashboard/composables';

const props = defineProps({
  currentUserId: { type: Number, required: true },
  firstUnreadId: { type: Number, default: null },
  isAnEmailChannel: { type: Boolean, default: false },
  inboxSupportsReplyTo: { type: Object, default: () => ({ incoming: false, outgoing: false }) },
  messages: { type: Array, default: () => [] },
});

const emit = defineEmits(['retry']);
const { t } = useI18n();

const allMessages = computed(() => {
  return useCamelCase(props.messages, {
    deep: true,
    stopPaths: ['content_attributes.translations'],
  });
});

const currentChat = useMapGetter('getSelectedChat');

const selectedMessageIds = reactive(new Set());
const isForwardModalOpen = ref(false);
const isForwarding = ref(false);

const selectedCount = computed(() => selectedMessageIds.size);

const isSelected = id => selectedMessageIds.has(id);
const toggle = id => {
  if (selectedMessageIds.has(id)) {
    selectedMessageIds.delete(id);
    return;
  }
  selectedMessageIds.add(id);
};
const clearSelection = () => selectedMessageIds.clear();

provide('messageSelection', {
  isSelected,
  toggle,
  hasSelection: () => selectedMessageIds.size > 0,
});

const fetchedReplyMessages = reactive(new Map());

const fetchReplyMessage = async (messageId, conversationId) => {
  if (fetchedReplyMessages.has(messageId)) {
    return fetchedReplyMessages.get(messageId);
  }
  try {
    const response = await MessageApi.getPreviousMessages({
      conversationId,
      before: messageId + 100,
      after: messageId - 100,
    });
    const messages = response.data?.payload || [];
    const targetMessage = messages.find(msg => msg.id === messageId);
    if (targetMessage) {
      const camelCaseMessage = useCamelCase(targetMessage);
      fetchedReplyMessages.set(messageId, camelCaseMessage);
      return camelCaseMessage;
    }
    fetchedReplyMessages.set(messageId, null);
    return null;
  } catch (error) {
    fetchedReplyMessages.set(messageId, null);
    return null;
  }
};

const shouldGroupWithNext = (index, searchList) => {
  if (index === searchList.length - 1) return false;
  const current = searchList[index];
  const next = searchList[index + 1];
  if (next.status === 'failed') return false;
  const nextSenderId = next.senderId ?? next.sender?.id;
  const currentSenderId = current.senderId ?? current.sender?.id;
  const hasSameSender = nextSenderId === currentSenderId;
  const nextMessageType = next.messageType;
  const currentMessageType = current.messageType;
  const areBothTemplates =
    nextMessageType === MESSAGE_TYPES.TEMPLATE &&
    currentMessageType === MESSAGE_TYPES.TEMPLATE;
  if (!hasSameSender || areBothTemplates) return false;
  if (currentMessageType !== nextMessageType) return false;
  return Math.floor(next.createdAt / 60) === Math.floor(current.createdAt / 60);
};

const getInReplyToMessage = parentMessage => {
  if (!parentMessage) return null;
  const inReplyToMessageId =
    parentMessage.contentAttributes?.inReplyTo ??
    parentMessage.content_attributes?.in_reply_to;
  if (!inReplyToMessageId) return null;
  let replyMessage = props.messages?.find(msg => msg.id === inReplyToMessageId);
  if (!replyMessage && currentChat.value?.messages) {
    replyMessage = currentChat.value.messages.find(msg => msg.id === inReplyToMessageId);
  }
  if (!replyMessage && fetchedReplyMessages.has(inReplyToMessageId)) {
    replyMessage = fetchedReplyMessages.get(inReplyToMessageId);
  }
  if (!replyMessage && currentChat.value?.id) {
    fetchReplyMessage(inReplyToMessageId, currentChat.value.id);
    return null;
  }
  return replyMessage ? useCamelCase(replyMessage) : null;
};

const openForwardModal = () => {
  if (selectedMessageIds.size === 0) return;
  isForwardModalOpen.value = true;
};

const contentTypeToExtension = contentType => {
  const map = {
    'image/jpeg': 'jpg',
    'image/jpg': 'jpg',
    'image/png': 'png',
    'image/webp': 'webp',
    'image/gif': 'gif',
    'video/mp4': 'mp4',
    'audio/mpeg': 'mp3',
    'audio/mp3': 'mp3',
    'audio/wav': 'wav',
    'application/pdf': 'pdf',
  };
  return map[contentType] || '';
};

const inferFilename = (attachment, response) => {
  const byExtension = attachment?.extension ? `attachment-${attachment.id}.${attachment.extension}` : '';
  if (byExtension) return byExtension;
  const url = attachment?.dataUrl || attachment?.data_url;
  if (url) {
    try {
      const pathname = new URL(url, window.location.origin).pathname;
      const name = pathname.split('/').filter(Boolean).pop();
      if (name && name.includes('.')) return name;
    } catch (e) {
      // ignore
    }
  }
  const contentType = response?.headers?.get?.('content-type') || '';
  const ext = contentTypeToExtension(contentType);
  return ext ? `attachment-${attachment?.id}.${ext}` : `attachment-${attachment?.id}`;
};

const downloadAttachmentsAsFiles = async attachments => {
  const items = Array.isArray(attachments) ? attachments : [];
  if (items.length === 0) return [];

  const files = [];
  for (const attachment of items) {
    const url = attachment?.dataUrl || attachment?.data_url;
    if (!url) continue;

    try {
      // Primer intento: sin credentials (funciona con S3 público y URLs directas)
      let response = await fetch(url);

      // Si falla, intentar con credentials de sesión (para URLs locales del servidor)
      if (!response.ok) {
        response = await fetch(url, { credentials: 'include' });
      }

      if (!response.ok) {
        console.error('Error descargando attachment, status:', response.status, url);
        continue;
      }

      const blob = await response.blob();

      if (!blob || blob.size === 0) {
        console.error('Blob vacío para attachment:', url);
        continue;
      }

      const filename = inferFilename(attachment, response);
      const file = new File([blob], filename, {
        type: blob.type || response.headers.get('content-type') || '',
      });
      files.push(file);
    } catch (e) {
      console.error('Error descargando attachment:', url, e);
    }
  }
  return files;
};

const forwardSelectedMessages = async destinationConversationId => {
  const destinationId = Number(destinationConversationId);
  if (!destinationId || selectedMessageIds.size === 0) return;

  isForwarding.value = true;
  try {
    const selectedMessages = allMessages.value
      .filter(m => selectedMessageIds.has(m.id))
      .sort((a, b) => {
        const t = (a.createdAt || 0) - (b.createdAt || 0);
        if (t !== 0) return t;
        return (a.id || 0) - (b.id || 0);
      });

    let successCount = 0;
    for (const message of selectedMessages) {
      const files = await downloadAttachmentsAsFiles(message.attachments);
      await MessageApi.create({
        conversationId: destinationId,
        message: message.content,
        private: message.private,
        contentAttributes: {},
        files,
      });
      successCount += 1;
    }

    useAlert(t('CONVERSATION.FORWARD.SUCCESS', { count: successCount }));
    isForwardModalOpen.value = false;
    clearSelection();
  } catch (e) {
    useAlert(t('CONVERSATION.FORWARD.ERROR'));
  } finally {
    isForwarding.value = false;
  }
};

const handleForwardConfirm = ({ conversationId }) => {
  forwardSelectedMessages(conversationId);
};

const toLocalDayKey = createdAt => {
  const d = new Date((createdAt || 0) * 1000);
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(
    d.getDate()
  ).padStart(2, '0')}`;
};

const formatLocalDayLabelShort = createdAt => {
  const d = new Date((createdAt || 0) * 1000);
  return new Intl.DateTimeFormat(undefined, {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  }).format(d);
};

const shouldShowDayDivider = (messages, index) => {
  if (index === 0) return true;
  return (
    toLocalDayKey(messages[index - 1]?.createdAt) !==
    toLocalDayKey(messages[index]?.createdAt)
  );
};
</script>

<template>
  <div class="bg-n-surface-1">
    <div
      v-if="selectedCount > 0"
      class="sticky top-0 z-20 flex items-center justify-between gap-2 px-4 py-2 border-b border-n-strong bg-n-surface-1"
    >
      <div class="text-sm font-medium text-n-slate-12">
        {{ t('CONVERSATION.FORWARD.SELECTED_COUNT', { count: selectedCount }) }}
      </div>
      <div class="flex items-center gap-2">
        <NextButton
          :label="t('CONVERSATION.FORWARD.FORWARD_BUTTON', { count: selectedCount })"
          icon="i-lucide-forward"
          primary
          sm
          :disabled="isForwarding"
          @click="openForwardModal"
        />
        <NextButton
          :label="t('CONVERSATION.FORWARD.CANCEL_SELECTION')"
          icon="i-lucide-x"
          slate
          sm
          outline
          :disabled="isForwarding"
          @click="clearSelection"
        />
      </div>
    </div>

    <ForwardMessagesModal
      v-model:show="isForwardModalOpen"
      :selected-count="selectedCount"
      :is-busy="isForwarding"
      @confirm="handleForwardConfirm"
    />

    <ul class="px-4 bg-n-surface-1">
      <slot name="beforeAll" />
      <template v-for="(message, index) in allMessages" :key="message.id">
        <li
          v-if="shouldShowDayDivider(allMessages, index)"
          class="flex justify-center my-3"
        >
          <span
            class="text-xs px-3 py-1 rounded-full bg-n-alpha-2 text-n-slate-11 border border-n-strong"
          >
            {{ formatLocalDayLabelShort(message.createdAt) }}
          </span>
        </li>
        <slot
          v-if="firstUnreadId && message.id === firstUnreadId"
          name="unreadBadge"
        />
        <Message
          v-bind="message"
          :is-email-inbox="isAnEmailChannel"
          :in-reply-to="getInReplyToMessage(message)"
          :group-with-next="shouldGroupWithNext(index, allMessages)"
          :inbox-supports-reply-to="inboxSupportsReplyTo"
          :current-user-id="currentUserId"
          data-clarity-mask="True"
          @retry="emit('retry', message)"
        />
      </template>
      <slot name="after" />
    </ul>
  </div>
</template>
