<template>
  <van-popup v-model:show="popupShow" position="bottom" round destroy-on-close>
    <div class="popup-body">
      <div class="popup-head">
        <div>
          <h3>更多操作</h3>
        </div>
      </div>

      <div class="theme-panel">
        <div class="theme-panel-head">
          <strong>界面主题</strong>
          <span>{{ currentThemeName }}</span>
        </div>
        <div class="theme-options">
          <button
            v-for="theme in themeOptions"
            :key="theme.key"
            class="theme-option"
            :class="{ active: currentTheme === theme.key }"
            type="button"
            @click="emit('set-theme', theme.key)"
          >
            <span class="theme-swatch" :style="{ background: theme.swatch }"></span>
            <span>
              <strong>{{ theme.name }}</strong>
              <em>{{ theme.caption }}</em>
            </span>
          </button>
        </div>
      </div>

      <div class="settings-grid">
        <button class="settings-item settings-item-feature" type="button" @click="emit('export')">
          <span class="settings-item-icon">
            <el-icon><Download /></el-icon>
          </span>
          <span>
            <strong>导出总数据</strong>
            <em>下载库房备份文件</em>
          </span>
        </button>

        <button class="settings-item settings-item-feature" type="button" @click="emit('open-import')">
          <span class="settings-item-icon">
            <el-icon><Upload /></el-icon>
          </span>
          <span>
            <strong>导入总数据</strong>
            <em>上传备份文件后恢复库房</em>
          </span>
        </button>

        <button class="settings-item settings-item-feature" type="button" @click="emit('open-recycle')">
          <span class="settings-item-icon">
            <el-icon><RefreshLeft /></el-icon>
          </span>
          <span>
            <strong>回收站</strong>
            <em>{{ recycleCount }} 个物品，一周内可恢复</em>
          </span>
        </button>

        <button class="settings-item settings-item-feature danger" type="button" @click="emit('reset')">
          <span class="settings-item-icon">
            <el-icon><Delete /></el-icon>
          </span>
          <span>
            <strong>重置数据</strong>
            <em>清除全部库房数据</em>
          </span>
        </button>
      </div>
    </div>
  </van-popup>
</template>

<script lang="ts" setup>
import { computed } from "vue";
import { Delete, Download, RefreshLeft, Upload } from "@element-plus/icons-vue";
import type { ThemeKey, ThemeOption } from "@/config/themes";

const props = defineProps<{
  show: boolean;
  themeOptions: ThemeOption[];
  currentTheme: ThemeKey;
  currentThemeName: string;
  recycleCount: number;
}>();

const emit = defineEmits<{
  "update:show": [value: boolean];
  "set-theme": [theme: ThemeKey];
  export: [];
  "open-import": [];
  "open-recycle": [];
  reset: [];
}>();

const popupShow = computed({
  get: () => props.show,
  set: (value: boolean) => emit("update:show", value),
});
</script>

<style lang="scss" scoped>
.theme-panel {
  margin-top: 20px;
}

.theme-panel-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 10px;
}

.theme-panel-head strong {
  color: var(--text-muted);
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.04em;
}

.theme-panel-head span {
  flex: 0 0 auto;
  color: var(--accent);
  font-size: 12px;
  font-weight: 600;
}

.theme-options {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
}

.theme-option {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 10px;
  min-width: 0;
  padding: 12px;
  border: 1px solid var(--divider);
  border-radius: 14px;
  background: var(--surface);
  text-align: left;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.theme-option:hover {
  border-color: var(--accent-border);
}

.theme-option.active {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px var(--accent-ring);
}

.theme-swatch {
  flex: 0 0 auto;
  width: 28px;
  height: 28px;
  border-radius: 999px;
  box-shadow: inset 0 0 0 1px rgba(17, 24, 39, 0.08);
}

.theme-option > span:last-child {
  min-width: 0;
}

.theme-option strong,
.theme-option em {
  display: block;
}

.theme-option strong {
  color: var(--text-strong);
  font-size: 14px;
  font-weight: 600;
}

.theme-option em {
  margin-top: 2px;
  color: var(--text-muted);
  font-size: 11px;
  font-style: normal;
  line-height: 1.3;
}

.settings-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
  margin-top: 20px;
}

.settings-item {
  padding: 14px;
  border: 1px solid var(--divider);
  border-radius: 14px;
  background: var(--surface);
  text-align: left;
  transition: border-color 0.2s ease, background-color 0.2s ease, transform 0.15s var(--ease);
}

.settings-item:hover {
  border-color: var(--accent-border);
  background: var(--surface-soft);
}

.settings-item:active {
  transform: scale(0.985);
}

.settings-item-feature {
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
}

.settings-item-feature.danger:hover {
  border-color: color-mix(in srgb, var(--danger-text) 30%, transparent);
  background: var(--danger-bg);
}

.settings-item span {
  display: block;
}

.settings-item strong {
  display: block;
  color: var(--text-strong);
  font-size: 14px;
  font-weight: 600;
}

.settings-item-feature.danger strong {
  color: var(--danger-text);
}

.settings-item-feature > span:last-child {
  min-width: 0;
  margin-top: 0;
}

.settings-item-feature em {
  display: block;
  margin-top: 3px;
  color: var(--text-muted);
  font-size: 12px;
  font-style: normal;
  line-height: 1.4;
}

@media (max-width: 720px) {
  .settings-grid {
    grid-template-columns: minmax(0, 1fr);
    gap: 0;
    border: 1px solid var(--divider);
    border-radius: 14px;
    overflow: hidden;
  }

  .settings-item {
    border: 0;
    border-radius: 0;
  }

  .settings-item + .settings-item {
    border-top: 1px solid var(--divider);
  }

  .settings-item:hover {
    border-color: var(--divider);
  }

  .settings-item:active {
    transform: none;
    background: var(--surface-soft);
  }
}
</style>
