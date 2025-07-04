<script setup>
/* global chrome */
import { ref, onMounted, computed, watch } from 'vue'
import { storeToRefs } from 'pinia'
import { useColorStore } from '../stores/ColorStore.js'
import ColorManager from '../components/ColorManager.vue'
import ColorPreview from '../components/ColorPreview.vue'

const colorStore = useColorStore()
const { backgroundColor, textColor } = storeToRefs(colorStore)

const isReady = ref(false)
const cuContentPrimary = ref('#2097f3')
const cuBackgroundPrimary = ref('#fe5722')
const theme = ref('system') // 'system', 'light', 'dark'

// Load theme preference from Chrome storage
chrome.storage.local.get(['theme'], (result) => {
  if (result.theme) {
    theme.value = result.theme
    applyTheme(theme.value)
  }
})

// Watch for theme changes and save to Chrome storage
watch(theme, (newTheme) => {
  chrome.storage.local.set({ theme: newTheme })
})
// const manualTheme = ref('light') // 'light' or 'dark'

// Load ClickUp theme variables from storage
chrome.storage.local.get(['cuContentPrimary', 'cuBackgroundPrimary'], (result) => {
  if (result.cuContentPrimary) {
    cuContentPrimary.value = result.cuContentPrimary
  }
  if (result.cuBackgroundPrimary) {
    cuBackgroundPrimary.value = result.cuBackgroundPrimary
  }
})

// Resolve a CSS variable (e.g. 'var(--cu-content-primary)') to real color
function resolveCssVariable(colorValue) {
  if (colorValue.startsWith('var(')) {
    const cssVarName = colorValue.match(/var\((--[^)]+)\)/)?.[1]
    if (cssVarName) {
      // Use stored value if available
      if (cssVarName === '--cu-content-primary') return cuContentPrimary.value
      if (cssVarName === '--cu-background-primary') return cuBackgroundPrimary.value
      return getComputedStyle(document.documentElement).getPropertyValue(cssVarName).trim()
    }
  }
  return colorValue
}

function applyTheme(newTheme) {
  let target = document.documentElement
  document.body.removeAttribute('data-theme')
  if (newTheme === 'system') {
    const isDark = window.matchMedia('(prefers-color-scheme: dark)').matches
    target.setAttribute('data-theme', isDark ? 'dark' : 'light')
    // Listen for system changes only in system mode
    setupSystemThemeWatcher()
  } else {
    target.setAttribute('data-theme', newTheme)
    // Remove system watcher if not in system mode
    if (mediaQueryList) {
      mediaQueryList.removeEventListener('change', handleSystemThemeChange)
      mediaQueryList = null
    }
  }
  // Log out the element and its computed background and color
  const body = document.body
  const bg = getComputedStyle(body).backgroundColor
  const color = getComputedStyle(body).color
  console.log('Theme applied to:', target, 'Body bg:', bg, 'Body color:', color)
}

// Watch for browser theme changes if theme is 'system'
let mediaQueryList = null
function setupSystemThemeWatcher() {
  if (mediaQueryList) {
    mediaQueryList.removeEventListener('change', handleSystemThemeChange)
  }
  mediaQueryList = window.matchMedia('(prefers-color-scheme: dark)')
  mediaQueryList.addEventListener('change', handleSystemThemeChange)
}
function handleSystemThemeChange(e) {
  if (theme.value === 'system') {
    document.documentElement.setAttribute('data-theme', e.matches ? 'dark' : 'light')
  }
}

watch(theme, (newTheme) => {
  console.log('Theme changed to:', newTheme)
  applyTheme(newTheme)
  chrome.storage.local.set({ umcTheme: newTheme })
})

onMounted(() => {
  // Default: inherit system theme
  theme.value = 'system'
  applyTheme('system')
  chrome.storage.local.get(
    [
      'backgroundColor',
      'textColor',
      'cuContentPrimary',
      'cuBackgroundPrimary',
      'umcTheme',
      'umcManualTheme',
    ],
    ({
      backgroundColor,
      textColor,
      cuContentPrimary: cuTxt,
      cuBackgroundPrimary: cuBg,
      umcTheme,
      // umcManualTheme,
    }) => {
      if (backgroundColor && textColor) {
        colorStore.setColors(backgroundColor, textColor)
      }
      if (cuTxt) cuContentPrimary.value = cuTxt
      if (cuBg) cuBackgroundPrimary.value = cuBg
      if (umcTheme && umcTheme !== 'system') {
        theme.value = umcTheme
        applyTheme(umcTheme)
      }
      setupSystemThemeWatcher()
      isReady.value = true
    },
  )
})

// Send message directly to the content script
function sendMessageToContentScript(bg, txt) {
  // Get all active tabs with the ClickUp URL
  chrome.tabs.query({ url: 'https://app.clickup.com/*' }, (tabs) => {
    if (tabs.length === 0) {
      console.log('No ClickUp tabs found')
      return
    }

    // Send message to all ClickUp tabs
    tabs.forEach((tab) => {
      chrome.tabs.sendMessage(
        tab.id,
        {
          action: 'updateColors',
          backgroundColor: bg,
          textColor: txt,
        },
        (response) => {
          // Handle the response from the content script
          if (chrome.runtime.lastError) {
            console.error('Error sending message:', chrome.runtime.lastError)
            return
          }

          if (response && response.success) {
            console.log('Colors updated in ClickUp tab:', tab.id)
          }
        },
      )
    })
  })
}

const handleColorsChanged = ({ backgroundColor: bg, textColor: txt }) => {
  colorStore.setColors(bg, txt)
  // Store in local storage for persistence
  chrome.storage.local.set({
    backgroundColor: bg,
    textColor: txt,
  })

  // Send direct message to content script for immediate update
  sendMessageToContentScript(bg, txt)
}

const previewBackground = computed(() => resolveCssVariable(backgroundColor.value))
const previewText = computed(() => resolveCssVariable(textColor.value))
</script>

<template>
  <main class="extension-wrapper">
    <div class="theme-selector-wrapper">
      <label for="theme-select">Appearance</label>
      <select id="theme-select" v-model="theme" @change="applyTheme(theme)" class="theme-selector">
        <option value="system">System</option>
        <option value="light">Light</option>
        <option value="dark">Dark</option>
      </select>
    </div>
    <!-- <h1>Choose your colors</h1> -->
    <div v-if="isReady" class="color-picker-wrapper">
      <ColorManager @colorsChanged="handleColorsChanged" />
      <ColorPreview :backgroundColor="previewBackground" :textColor="previewText" />
    </div>
    <div v-else class="loading-wrapper flex items-center justify-center h-32 text-lg font-semibold">
      loading...
    </div>
  </main>
</template>

<style scoped>
.extension-wrapper {
  width: var(--full-width);
}
.extension-wrapper h1 {
  margin: var(--spacing-base) 0 var(--spacing-sm) 0;
  font-size: var(--font-size-base);
}
.theme-selector-wrapper {
  display: var(--flex-center);
  align-items: var(--align-center);
  justify-content: var(--justify-between);
  width: var(--full-width);
  margin: 0 auto;
  padding-block: var(--spacing-md-lg);
  padding-inline: var(--spacing-lg);
  border: var(--border-default);
  border-radius: var(--border-radius-sm);
}
.theme-selector {
  padding-block: var(--spacing-sm);
  background-color: var(--bg-color);
  color: var(--text-color);
  border-color: var(--border-color);
  border-radius: var(--border-radius-sm);
}

.theme-selector:hover {
  opacity: var(--hover-opacity);
}
.color-picker-wrapper {
  position: relative;
  margin-top: var(--spacing-xl);
  padding: var(--spacing-lg);
  border: var(--border-default);
  border-radius: var(--border-radius-sm);
}
.color-picker-wrapper::before {
  content: 'Choose your colors';
  position: absolute;
  width: var(--width-12);
  height: var(--spacing-2xl);
  top: calc(var(--spacing-base) * -1);
  left: var(--spacing-base);
  right: 0;
  bottom: 0;
  z-index: 10;
  font-size: var(--font-size-base);
  font-weight: var(--font-weight-semibold);
  text-align: center;
  background-color: var(--color-dark-gray);
}
</style>
