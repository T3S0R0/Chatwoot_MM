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

/**
 * Props definition for the component
 * @typedef {Object} Props
 * @property {Array} readMessages - Array of read messages
 * @property {Array} unReadMessages - Array of unread messages
 * @property {Number} currentUserId - ID of the current user
 * @property {Boolean} isAnEmailChannel - Whether this is an email channel
 * @property {Object} inboxSupportsReplyTo - Inbox reply support configuration
 * @property {Array} messages - Array of all messages [These are not in camelcase]
 */
const props = defineProps({
  currentUserId: {
    type: Number,
    required: true,
  },
  firstUnreadId: {
    type: Number,
    default: null,
  },
  isAnEmailChannel: {
    type: Boolean,
    default: false,
  },
  inboxSupportsReplyTo: {
    type: Object,
    default: () => ({ incoming: false, outgoing: false }),
  },
  messages: {
    type: Array,
    default: () => [],
  },
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

// Cache for fetched reply messages to avoid duplicate API calls
const fetchedReplyMessages = reactive(new Map());

/**
 * Fetches a specific message from the API by trying to get messages around it
 * @param {number} messageId - The ID of the message to fetch
 * @param {number} conversationId - The ID of the conversation
 * @returns {Promise<Object|null>} - The fetched message or null if not found/error
 */
const fetchReplyMessage = async (messageId, conversationId) => {
  // Return cached result if already fetched
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

    // Cache null result to avoid repeated API calls
    fetchedReplyMessages.set(messageId, null);
    return null;
  } catch (error) {
    fetchedReplyMessages.set(messageId, null);
    return null;
  }
};

/**
 * Determines if a message should be grouped with the next message
 * @param {Number} index - Index of the current message
 * @param {Array} searchList - Array of messages to check
 * @returns {Boolean} - Whether the message should be grouped with next
 */
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

  // Check if messages are in the same minute by rounding down to nearest minute
  return Math.floor(next.createdAt / 60) === Math.floor(current.createdAt / 60);
};

/**
 * Gets the message that was replied to
 * @param {Object} parentMessage - The message containing the reply reference
 * @returns {Object|null} - The message being replied to, or null if not found
 */
const getInReplyToMessage = parentMessage => {
  if (!parentMessage) return null;

  const inReplyToMessageId =
    parentMessage.contentAttributes?.inReplyTo ??
    parentMessage.content_attributes?.in_reply_to;

  if (!inReplyToMessageId) return null;

  // Try to find in current messages first
  let replyMessage = props.messages?.find(msg => msg.id === inReplyToMessageId);

  // Then try store messages
  if (!replyMessage && currentChat.value?.messages) {
    replyMessage = currentChat.value.messages.find(
      msg => msg.id === inReplyToMessageId
    );
  }

  // Then check fetch cache
  if (!replyMessage && fetchedReplyMessages.has(inReplyToMessageId)) {
    replyMessage = fetchedReplyMessages.get(inReplyToMessageId);
  }

  // If still not found and we have conversation context, fetch it
  if (!replyMessage && currentChat.value?.id) {
    fetchReplyMessage(inReplyToMessageId, currentChat.value.id);
    return null; // Let UI handle loading state
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

/**
 * Converts an Active Storage redirect URL to a proxy URL so Rails serves
 * the file directly instead of redirecting the browser to S3.
 * This avoids CORS errors because the request never leaves our own origin.
 *
 * /rails/active_storage/blobs/redirect/<signed_id>/filename
 *   → /rails/active_storage/blobs/proxy/<signed_id>/filename
 *
 * If the URL is already a proxy URL or is an external URL (e.g. a direct
 * S3 URL without an Active Storage path), it is returned unchanged.
 */
const toProxyUrl = rawUrl => {
  if (!rawUrl) return rawUrl;
  try {
    const url = new URL(rawUrl, window.location.origin);
    // Only rewrite same-origin Active Storage redirect URLs
    if (url.origin !== window.location.origin) return rawUrl;
    return url.pathname.includes('/blobs/redirect/')
      ? rawUrl.replace('/blobs/redirect/', '/blobs/proxy/')
      : rawUrl;
  } catch {
    return rawUrl;
  }
};

const downloadAttachmentsAsFiles = async attachments => {
  const items = Array.isArray(attachments) ? attachments : [];
  if (items.length === 0) return [];

  const files = [];
  for (const attachment of items) {
    const rawUrl = attachment?.dataUrl || attachment?.data_url;
    if (!rawUrl) continue;

    try {
      // Use proxy URL so Rails streams the file — no S3 redirect, no CORS
      const url = toProxyUrl(rawUrl);
      const response = await fetch(url, { credentials: 'same-origin' });
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const blob = await response.blob();
      const filename = inferFilename(attachment, response);
      files.push(new File([blob], filename, {
        type: blob.type || response.headers.get('content-type') || '',
      }));
    } catch (err) {
      console.warn('[ForwardMessage] Could not download attachment:', rawUrl, err);
      // Skip this attachment rather than failing the entire forward
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
