<template>
  <transition name="fade-slide" appear>
    <div
      class="fixed left-1/2 transform -translate-x-1/2 bg-blue-100 text-blue-800 px-4 py-2 rounded shadow text-sm z-50"
    >
      <div
        v-if="showInstallPrompt"
        :class="['flex items-center gap-2 min-w-[320px]', !isOnline ? 'top-[56px]' : 'top-2']"
        :style="!isOnline ? 'margin-top:4px;' : ''"
      >
        <span>Install this app for a better experience!</span>
        <button
          @click="$emit('install')"
          class="bg-white text-blue-800 px-3 py-1 rounded"
          v-if="deviceType !== 'Apple'"
        >
          Install
        </button>
      </div>
      <span v-if="deviceType === 'Apple'">
        <b>iPhone:</b> Open in <b>Safari</b>, tap <b>Share</b>
        <span aria-label="Share">
          <Share class="inline-block ml-1" />
          and then <b>Add to Home Screen</b>.
        </span>
      </span>
      <span v-else-if="deviceType === 'Android'">
        <b>Android:</b> Use <b>Chrome</b> and tap <b>Install</b> or <b>Add to Home Screen</b> in the
        browser menu.
      </span>
      <span v-else>
        <b>{{ browser }}</b
        >: Click <b>Install</b> to add this app to your device.
      </span>
    </div>
  </transition>
</template>

<script lang="ts" setup>
import { defineProps, defineEmits, computed } from 'vue'
import Share from './Share.vue'
const props = defineProps<{
  showInstallPrompt: boolean
  isOnline: boolean
}>()

defineEmits(['install'])

const userAgent = navigator.userAgent || (window as any).opera

const deviceType = computed(() => {
  if (/iPhone|iPad|iPod/i.test(userAgent)) return 'Apple'
  if (/Android/i.test(userAgent)) return 'Android'
  return 'Desktop'
})

const browser = computed(() => {
  if (/CriOS/i.test(userAgent)) return 'Chrome (iOS)'
  if (/Chrome/i.test(userAgent)) return 'Chrome'
  if (/Safari/i.test(userAgent) && !/Chrome/i.test(userAgent)) return 'Safari'
  if (/Firefox/i.test(userAgent)) return 'Firefox'
  if (/Edg/i.test(userAgent)) return 'Edge'
  if (/OPR|Opera/i.test(userAgent)) return 'Opera'
  return 'Browser'
})
</script>
