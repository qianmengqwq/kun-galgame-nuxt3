<script setup lang="ts">
import dayjs from 'dayjs'
import { getMessageI18n } from './getMessageI18n'
import type { Message } from '~/types/api/message'

const props = defineProps<{
  message: Message[]
  refresh: () => {}
}>()

const localePath = useLocalePath()
const { locale } = useI18n()

const activeMessage = ref<number[]>([])
const messageMap = reactive(new Map<number, string | null>())

const isShowMoreOperation = (mid: number) => {
  return activeMessage.value.includes(mid)
}

const handelClickShowMoreOperation = async (mid: number) => {
  if (!isShowMoreOperation(mid)) {
    activeMessage.value.push(mid)

    if (messageMap.get(mid)) {
      return
    }
    const result = await $fetch(`/api/message`, {
      method: 'GET',
      query: { mid },
      watch: false,
      ...kungalgameResponseHandler
    })
    messageMap.set(mid, result)
  } else {
    activeMessage.value = activeMessage.value.filter((item) => item !== mid)
  }
}

const handleGetMessageDetail = (message: Message) => {
  return computed(() => {
    const messageContent = markdownToText(messageMap.get(message.mid) ?? '')
    return `${getMessageI18n(locale.value as Language, message)} => ${messageContent}`
  }).value
}

const handleMarkAsRead = async (mid: number) => {
  const result = await $fetch(`/api/message/read`, {
    method: 'PUT',
    query: { mid },
    watch: false,
    ...kungalgameResponseHandler
  })

  if (result) {
    props.refresh()
    useMessage(10105, 'success')
  }
}

const handleDeleteMessage = async (mid: number) => {
  const result = await $fetch(`/api/message/delete`, {
    method: 'DELETE',
    query: { mid },
    watch: false,
    ...kungalgameResponseHandler
  })

  if (result) {
    props.refresh()
    useMessage(10106, 'success')
  }
}
</script>

<template>
  <div class="container">
    <div
      class="message"
      :class="msg.status === 'read' ? 'message-read' : ''"
      v-for="(msg, index) in props.message"
      :key="index"
    >
      <div class="top">
        <div class="status">
          <span class="unread" v-if="msg.status === 'unread'">
            <Icon class="icon" name="lucide:info" />
          </span>
          <span class="read" v-if="msg.status === 'read'">
            <Icon class="icon" name="lucide:check-check" />
          </span>
        </div>

        <span class="time">
          {{ dayjs(msg.time).format('MM-D-YYYY - HH:mm:ss') }}
        </span>
      </div>

      <div class="content" v-html="getMessageI18n(locale as Language, msg)" />

      <div class="bottom">
        <NuxtLinkLocale
          v-if="msg.tid"
          class="link"
          :to="msg.tid > 0 ? `/topic/${msg.tid}` : `/galgame/${-msg.tid}`"
        >
          <Icon class="icon" name="lucide:external-link" />
        </NuxtLinkLocale>
        <span
          class="more-btn"
          :class="isShowMoreOperation(msg.mid) ? 'more-btn-active' : ''"
          @click="handelClickShowMoreOperation(msg.mid)"
        >
          <Icon class="icon" name="lucide:chevron-right" />
        </span>
      </div>

      <div
        class="more"
        v-if="isShowMoreOperation(msg.mid) && activeMessage.includes(msg.mid)"
      >
        <pre class="detail">{{ handleGetMessageDetail(msg) }}</pre>

        <span
          @click="navigateTo(localePath(`/kungalgamer/${msg.senderUid}/info`))"
        >
          {{ `${$t('header.message.goto')} ${msg.senderName}` }}
        </span>
        <span v-if="msg.status === 'unread'" @click="handleMarkAsRead(msg.mid)">
          {{ $t('header.message.read') }}
        </span>
        <span @click="handleDeleteMessage(msg.mid)">
          {{ $t('header.message.delete') }}
        </span>
      </div>
    </div>

    <slot />
  </div>
</template>

<style lang="scss" scoped>
.container {
  height: 100%;
  padding: 10px;
}

.message {
  margin-top: 10px;
  padding: 10px;
  border-radius: 5px;
  border: 2px solid var(--kungalgame-pink-4);

  .top {
    display: flex;
    justify-content: space-between;
    align-items: center;

    .status {
      .unread {
        color: var(--kungalgame-pink-4);
      }

      .read {
        color: var(--kungalgame-blue-5);
      }
    }

    .time {
      font-size: 13px;
    }
  }

  .content {
    margin-top: 10px;
    margin-bottom: 10px;
    word-break: break-all;

    :deep(a) {
      color: var(--kungalgame-blue-5);

      &:hover {
        text-decoration: underline;
      }
    }
  }

  .bottom {
    display: flex;
    justify-content: space-between;
  }
}

.link {
  cursor: pointer;
  color: var(--kungalgame-pink-4);
}

.more-btn {
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  transition: transform 0.2s;

  &:hover {
    color: var(--kungalgame-blue-5);
  }
}

.more {
  margin-top: 10px;
  display: flex;
  flex-direction: column;
  justify-content: center;

  span {
    cursor: pointer;
    padding: 5px;
    border-radius: 5px;
    font-size: 15px;

    &:hover {
      background-color: var(--kungalgame-trans-blue-1);
    }
  }
}

.detail {
  margin: 0;
  padding: 5px;
  border-left: 4px solid var(--kungalgame-blue-5);
  word-break: break-word;
  white-space: pre-wrap;
  font-family: inherit;
}

.more-btn-active {
  transform: rotate(90deg);
  transition: transform 0.2s;
}

.message-read {
  border: 1px solid var(--kungalgame-trans-blue-4);

  .link {
    color: var(--kungalgame-blue-5);
  }
}
</style>
