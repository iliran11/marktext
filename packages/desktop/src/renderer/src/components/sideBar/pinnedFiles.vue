<template>
  <div class="pinned-files">
    <div class="title">
      <el-icon
        class="icon-arrow"
        :class="{ fold: !expanded }"
        :size="12"
        @click.stop="expanded = !expanded"
      >
        <ArrowRight />
      </el-icon>
      <span
        class="default-cursor text-overflow"
        @click.stop="expanded = !expanded"
      >{{ t('sideBar.tree.pinnedFiles') }}</span>
      <icon-tooltip
        v-if="currentFile?.pathname"
        :content="t('sideBar.tree.pinCurrentFile')"
      >
        <button
          type="button"
          class="plain-button"
          :aria-label="t('sideBar.tree.pinCurrentFile')"
          @click.stop="preferencesStore.PIN_FILE(currentFile.pathname)"
        >
          <el-icon :size="14">
            <Plus />
          </el-icon>
        </button>
      </icon-tooltip>
    </div>
    <div
      v-show="expanded"
      class="pinned-files-list"
    >
      <div
        v-if="!rows.length"
        class="empty"
      >
        {{ t('sideBar.tree.noPinnedFiles') }}
      </div>
      <div
        v-for="row of rows"
        :key="row.pathname"
        class="pinned-file"
        :class="{ active: isActive(row.pathname) }"
        :title="row.pathname"
        @click="openPinned(row.pathname)"
      >
        <span class="location">{{ row.location }}</span>
        <span class="name">{{ row.filename }}</span>
        <button
          type="button"
          class="plain-button unpin-icon"
          :aria-label="t('sideBar.tree.unpinFile')"
          @click.stop="preferencesStore.UNPIN_FILE(row.pathname)"
        >
          <el-icon :size="10">
            <Close />
          </el-icon>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue'
import { storeToRefs } from 'pinia'
import { useI18n } from 'vue-i18n'
import { ArrowRight, Close, Plus } from '@element-plus/icons-vue'
import { useEditorStore } from '@/store/editor'
import { usePreferencesStore } from '@/store/preferences'
import IconTooltip from '@/components/iconTooltip/index.vue'

const { t } = useI18n()
const editorStore = useEditorStore()
const preferencesStore = usePreferencesStore()
const { currentFile, tabs } = storeToRefs(editorStore)
const { pinnedFiles } = storeToRefs(preferencesStore)

const expanded = ref(true)

const homeDir = window.electron.process.env.HOME ?? ''

const toHomeRelative = (dirname: string): string => {
  if (!homeDir) return dirname
  if (dirname === homeDir) return '~'
  const prefix = homeDir.endsWith(window.path.sep) ? homeDir : homeDir + window.path.sep
  return dirname.startsWith(prefix) ? `~${window.path.sep}${dirname.slice(prefix.length)}` : dirname
}

const rows = computed(() =>
  pinnedFiles.value.map((pathname) => ({
    pathname,
    filename: window.path.basename(pathname),
    location: toHomeRelative(window.path.dirname(pathname))
  }))
)

const isActive = (pathname: string): boolean =>
  !!currentFile.value?.pathname && window.fileUtils.isSamePathSync(currentFile.value.pathname, pathname)

// Pinned files open as regular tabs, not preview tabs, since they are deliberately chosen.
const openPinned = (pathname: string): void => {
  const openedTab = tabs.value.find((f) => window.fileUtils.isSamePathSync(f.pathname, pathname))
  if (openedTab) {
    editorStore.UPDATE_CURRENT_FILE(openedTab)
    return
  }
  window.electron.ipcRenderer.send('mt::open-file', pathname, { preview: false })
}
</script>

<style scoped>
.pinned-files {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
}

.pinned-files > .title {
  height: 30px;
  line-height: 30px;
  font-size: 14px;
  padding: 0 15px;
  display: flex;
  align-items: center;
}

.pinned-files > .title > span {
  flex: 1;
}

.pinned-files > .title > button {
  color: var(--sideBarColor);
  margin-left: 8px;
  clip-path: inset(50%);
}
.pinned-files > .title:hover > button,
.pinned-files > .title:focus-within > button {
  clip-path: none;
}
.pinned-files > .title > button:hover {
  color: var(--highlightThemeColor);
}

.icon-arrow {
  margin-right: 5px;
  transition: transform 0.25s ease-out;
  transform: rotate(90deg);
  color: var(--sideBarTextColor);
  cursor: pointer;
}
.icon-arrow.fold {
  transform: rotate(0);
}
.default-cursor {
  cursor: pointer;
}

.pinned-files-list {
  max-height: 240px;
  overflow-x: hidden;
  overflow-y: auto;
}

.empty {
  padding: 0 15px 6px 35px;
  font-size: 12px;
  color: var(--sideBarTextColor);
}

.pinned-file {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
  column-gap: 8px;
  align-items: center;
  height: 28px;
  line-height: 28px;
  padding: 0 28px 0 35px;
  position: relative;
  cursor: default;
  color: var(--sideBarColor);
  & > span {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  & > .location {
    font-size: 12px;
    color: var(--sideBarTextColor);
  }
  & > .unpin-icon {
    position: absolute;
    top: 4px;
    right: 6px;
    width: 20px;
    height: 20px;
    border-radius: 4px;
    opacity: 0;
    transition: opacity 0.15s ease-in-out;
  }
  &:hover > .unpin-icon,
  & > .unpin-icon:focus-visible {
    opacity: 1;
  }
  & > .unpin-icon:hover {
    color: var(--focusColor);
    background-color: var(--floatHoverColor);
  }
  &:hover {
    background: var(--sideBarItemHoverBgColor);
  }
}
.pinned-file.active > .name {
  color: var(--highlightThemeColor);
}
</style>
