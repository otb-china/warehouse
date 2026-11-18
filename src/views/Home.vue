<template>
  <div class="home-page" :style="themeStyle">
    <div class="sticky-panel">
      <header class="page-header">
        <div class="header-title">
          <h1>库房</h1>
          <span class="item-count" aria-live="polite">共 {{ filteredItems.length }} 项</span>
        </div>
        <div class="header-actions">
          <button v-if="showScrollTop" class="header-icon" type="button" aria-label="返回顶部" @click="scrollToTop">
            <el-icon><Top /></el-icon>
          </button>
          <button class="header-icon" type="button" aria-label="设置" @click="settingsPopup = true">
            <el-icon><Setting /></el-icon>
          </button>
        </div>
      </header>

      <section class="search-section" aria-label="搜索和新增">
        <label class="search-box">
          <el-icon><Search /></el-icon>
          <input ref="searchInput" v-model.trim="keyword" type="search" aria-label="搜索库房物品" placeholder="搜索库房物品" />
        </label>
        <button class="create-button" type="button" @click="openEditor()">
          <el-icon><Plus /></el-icon>
          新增
        </button>
      </section>
    </div>

    <main class="warehouse-view">
      <section v-if="filteredItems.length" class="waterfall-list" aria-label="库房物品列表">
        <article
          v-for="(item, index) in filteredItems"
          :key="item.id"
          class="warehouse-card"
          :class="`tone-${index % 5}`"
          @click="openEditor(item)"
        >
          <button class="delete-btn" type="button" aria-label="移入回收站" @click.stop="removeItem(item.id)">×</button>
          <h2>{{ item.name }}</h2>
          <p class="quantity-line">{{ formatQuantity(item.quantity) }}<span class="quantity-unit">{{ item.unit }}</span></p>
          <time class="updated-date" :datetime="item.updatedAt">更新 {{ dayjs(item.updatedAt).format(DATE_FORMAT) }}</time>
        </article>
      </section>

      <div v-else class="empty-state">
        <strong>{{ keyword ? "没有匹配物品" : "还没有物品" }}</strong>
        <span>{{ keyword ? "换个关键词试试。" : "点击新增创建第一个库房物品。" }}</span>
      </div>
    </main>

    <van-popup v-model:show="editorPopup" class="edit-popup" round destroy-on-close>
      <form class="simple-editor" @submit.prevent="saveItem">
        <div class="editor-title">
          <strong>{{ editingItemId ? "编辑物品" : "新增物品" }}</strong>
          <span>填写名称、数量和单位</span>
        </div>

        <label class="field-row">
          <span>名称</span>
          <input v-model.trim="itemForm.name" class="simple-input" placeholder="例如：矿泉水" />
        </label>

        <div class="field-row">
          <span>数量</span>
          <div class="quantity-stepper">
            <button
              class="stepper-btn"
              type="button"
              aria-label="减少数量"
              :disabled="normalizeQuantity(itemForm.quantity) <= 0"
              @click="changeQuantity(-1)"
            >
              −
            </button>
            <input
              v-model.number="itemForm.quantity"
              class="simple-input stepper-input"
              aria-label="数量"
              inputmode="decimal"
              type="number"
              min="0"
              step="0.01"
            />
            <button class="stepper-btn" type="button" aria-label="增加数量" @click="changeQuantity(1)">+</button>
          </div>
        </div>

        <div class="field-row">
          <span>单位</span>
          <div class="unit-grid" role="radiogroup" aria-label="单位">
            <button
              v-for="unit in units"
              :key="unit"
              class="unit-option"
              :class="{ active: itemForm.unit === unit }"
              type="button"
              role="radio"
              :aria-checked="itemForm.unit === unit"
              @click="itemForm.unit = unit"
            >
              {{ unit }}
            </button>
          </div>
        </div>

        <div class="simple-actions">
          <button class="cancel-btn" type="button" @click="editorPopup = false">取消</button>
          <button class="confirm-btn" type="submit">保存</button>
        </div>
      </form>
    </van-popup>

    <SettingsPopup
      v-model:show="settingsPopup"
      :theme-options="themeOptions"
      :current-theme="currentTheme"
      :current-theme-name="currentThemeOption.name"
      :recycle-count="deletedItems.length"
      @set-theme="setTheme"
      @export="exportAllData"
      @open-import="openImportExport"
      @open-recycle="openRecycleBin"
      @reset="resetAllData"
    />

    <van-popup v-model:show="recyclePopup" position="bottom" round destroy-on-close>
      <div class="popup-body recycle-popup">
        <div class="popup-head">
          <div>
            <h3>回收站</h3>
            <p>移除的物品保留 7 天，可在这里恢复。</p>
          </div>
        </div>

        <div v-if="deletedItems.length" class="recycle-list">
          <div v-for="item in deletedItems" :key="item.id" class="recycle-item">
            <div>
              <strong>{{ item.name }}</strong>
              <span>{{ formatQuantity(item.quantity) }}{{ item.unit }} · {{ getRecycleDaysLeft(item) }} 天后清除</span>
            </div>
            <div class="recycle-actions">
              <button type="button" @click="restoreDeletedItem(item.id)">恢复</button>
              <button class="danger" type="button" @click="purgeDeletedItem(item.id)">删除</button>
            </div>
          </div>
        </div>

        <div v-else class="empty-state compact">
          <strong>回收站为空</strong>
          <span>移除的物品会在这里保留一周。</span>
        </div>
      </div>
    </van-popup>

    <ImportDataPopup
      v-model:show="importExportInfo.show"
      :has-local-data="hasImportLocalData"
      :summary="importExportSummary"
      :file-name="importInfo.fileName"
      :has-file="Boolean(importInfo.dataStr)"
      @file-loaded="onImportFileLoaded"
      @file-error="showToast"
      @import="importData"
    />
  </div>
</template>

<script lang="ts" setup>
import { computed, onMounted, onUnmounted, reactive, ref, watch } from "vue";
import dayjs from "dayjs";
import { showConfirmDialog, showToast } from "vant";
import { Plus, Search, Setting, Top } from "@element-plus/icons-vue";
import ImportDataPopup from "@/components/ImportDataPopup.vue";
import SettingsPopup from "@/components/SettingsPopup.vue";
import { DEFAULT_THEME, isThemeKey, themeOptions } from "@/config/themes";
import { LStorage } from "@/utils/localStorage.ts";
import type { ThemeKey } from "@/config/themes";
import type { DeletedWarehouseItem, WarehouseBackupData, WarehouseItem } from "@/types/tool";

const SCROLL_TOP_THRESHOLD = 240;
const DATE_FORMAT = "YYYY-MM-DD";
const RECYCLE_KEEP_DAYS = 7;
const RECYCLE_KEEP_MS = RECYCLE_KEEP_DAYS * 24 * 60 * 60 * 1000;
const units = ["个", "只", "支", "根", "袋", "瓶", "条", "米", "张", "箱", "盒", "包", "件", "卷", "套", "片"] as const;
type Unit = typeof units[number];

const showScrollTop = ref(false);
const searchInput = ref<HTMLInputElement | null>(null);
const currentTheme = ref<ThemeKey>(DEFAULT_THEME);
const items = ref<WarehouseItem[]>([]);
const deletedItems = ref<DeletedWarehouseItem[]>([]);
const keyword = ref("");
const settingsPopup = ref(false);
const recyclePopup = ref(false);
const editorPopup = ref(false);
const editingItemId = ref("");
const itemForm = reactive({ name: "", quantity: 1, unit: "个" as Unit });
const importExportInfo = ref({ show: false });
const importInfo = ref({ dataStr: "", fileName: "" });

const itemStorage = LStorage.new("warehouseItems");
const recycleStorage = LStorage.new("warehouseRecycleBin");
const themeStorage = LStorage.new("warehouseTheme");

const currentThemeOption = computed(() => {
  return themeOptions.find((theme) => theme.key === currentTheme.value) || themeOptions[0];
});
const themeStyle = computed(() => currentThemeOption.value.variables);
const filteredItems = computed(() => {
  const query = keyword.value.trim().toLowerCase();
  if (!query) return items.value;
  return items.value.filter((item) => item.name.toLowerCase().includes(query));
});
const importExportSummary = computed(() => `${items.value.length} 个在库物品，${deletedItems.value.length} 个回收站物品`);
const hasImportLocalData = computed(() => items.value.length > 0 || deletedItems.value.length > 0);

watch(items, saveItems, { deep: true });
watch(deletedItems, saveDeletedItems, { deep: true });
watch(themeStyle, (variables) => {
  Object.entries(variables).forEach(([name, value]) => document.documentElement.style.setProperty(name, value));
}, { immediate: true });

function init() {
  const storedTheme = themeStorage.getter();
  currentTheme.value = isThemeKey(storedTheme) ? storedTheme : DEFAULT_THEME;
  items.value = normalizeItems(itemStorage.getter());
  deletedItems.value = purgeExpiredDeletedItems(normalizeDeletedItems(recycleStorage.getter()));
}

function createItem(name: string, quantity: number, unit: string): WarehouseItem {
  return {
    id: createId("item"),
    name,
    quantity,
    unit,
    updatedAt: new Date().toISOString(),
  };
}

function createId(prefix = "id") {
  if (window.crypto?.randomUUID) return `${prefix}-${window.crypto.randomUUID()}`;
  return `${prefix}-${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

function normalizeItems(value: unknown): WarehouseItem[] {
  if (!Array.isArray(value)) return [];
  return value
    .filter((item): item is Partial<WarehouseItem> => Boolean(item && typeof item === "object"))
    .map((item) => ({
      id: typeof item.id === "string" && item.id ? item.id : createId("item"),
      name: typeof item.name === "string" && item.name.trim() ? item.name.trim() : "未命名物品",
      quantity: normalizeQuantity(item.quantity),
      unit: isUnit(item.unit) ? item.unit : "个",
      updatedAt: typeof item.updatedAt === "string" && dayjs(item.updatedAt).isValid()
        ? item.updatedAt
        : new Date().toISOString(),
    }));
}

function normalizeDeletedItems(value: unknown): DeletedWarehouseItem[] {
  if (!Array.isArray(value)) return [];
  return normalizeItems(value).map((item, index) => {
    const raw = value[index] as Partial<DeletedWarehouseItem>;
    return {
      ...item,
      deletedAt: typeof raw.deletedAt === "string" ? raw.deletedAt : new Date().toISOString(),
    };
  });
}

function normalizeQuantity(value: unknown) {
  const quantity = Number(value);
  return Number.isFinite(quantity) && quantity >= 0 ? quantity : 0;
}

function changeQuantity(delta: number) {
  const next = normalizeQuantity(itemForm.quantity) + delta;
  itemForm.quantity = Math.max(0, Number(next.toFixed(2)));
}

function mergeItemsByName<T extends WarehouseItem>(localItems: T[], importedItems: T[]): T[] {
  const merged: T[] = [];
  const indexes = new Map<string, number>();
  const usedIds = new Set<string>();

  for (const item of [...localItems, ...importedItems]) {
    const index = indexes.get(item.name);
    if (index !== undefined) {
      const existing = merged[index]!;
      if (new Date(item.updatedAt).getTime() > new Date(existing.updatedAt).getTime()) {
        merged[index] = { ...item, id: existing.id };
      }
      continue;
    }

    const id = usedIds.has(item.id) ? createId("item") : item.id;
    indexes.set(item.name, merged.length);
    usedIds.add(id);
    merged.push(id === item.id ? item : { ...item, id });
  }

  return merged;
}

function isUnit(value: unknown): value is Unit {
  return typeof value === "string" && (units as readonly string[]).includes(value);
}

function saveItems() {
  if (items.value.length) {
    itemStorage.setter(items.value);
  } else {
    itemStorage.remove();
  }
}

function saveDeletedItems() {
  if (deletedItems.value.length) {
    recycleStorage.setter(deletedItems.value);
  } else {
    recycleStorage.remove();
  }
}

function setTheme(theme: ThemeKey) {
  currentTheme.value = theme;
  themeStorage.setter(theme);
}

function openEditor(item?: WarehouseItem) {
  editingItemId.value = item?.id || "";
  itemForm.name = item?.name || "";
  itemForm.quantity = item?.quantity ?? 1;
  itemForm.unit = isUnit(item?.unit) ? item.unit : "个";
  editorPopup.value = true;
}

function saveItem() {
  const name = itemForm.name.trim();
  const quantity = normalizeQuantity(itemForm.quantity);
  if (!name) {
    showToast("请填写名称");
    return;
  }

  if (editingItemId.value) {
    items.value = items.value.map((item) => item.id === editingItemId.value ? {
      ...item,
      name,
      quantity,
      unit: itemForm.unit,
      updatedAt: new Date().toISOString(),
    } : item);
  } else {
    items.value = [createItem(name, quantity, itemForm.unit), ...items.value];
  }

  editorPopup.value = false;
  showToast("已保存");
}

function removeItem(id: string) {
  const target = items.value.find((item) => item.id === id);
  if (!target) return;
  showConfirmDialog({
    title: "移入回收站",
    message: `确认将“${target.name}”移入回收站吗？`,
    width: "280px",
  }).then(() => {
    const deletedItem = {
      ...target,
      deletedAt: new Date().toISOString(),
    };
    items.value = items.value.filter((item) => item.id !== id);
    deletedItems.value = [deletedItem, ...deletedItems.value.filter((item) => item.id !== id)];
    showToast("已移入回收站");
  }).catch(() => {});
}

function openRecycleBin() {
  deletedItems.value = purgeExpiredDeletedItems(deletedItems.value);
  settingsPopup.value = false;
  recyclePopup.value = true;
}

function purgeExpiredDeletedItems(list: DeletedWarehouseItem[]) {
  const now = Date.now();
  return list.filter((item) => now - new Date(item.deletedAt).getTime() < RECYCLE_KEEP_MS);
}

function getRecycleDaysLeft(item: DeletedWarehouseItem) {
  const deletedTime = new Date(item.deletedAt).getTime();
  const leftMs = Math.max(0, RECYCLE_KEEP_MS - (Date.now() - deletedTime));
  return Math.max(1, Math.ceil(leftMs / (24 * 60 * 60 * 1000)));
}

function restoreDeletedItem(id: string) {
  const deletedItem = deletedItems.value.find((item) => item.id === id);
  if (!deletedItem) return;
  const { deletedAt: _deletedAt, ...restoredItem } = deletedItem;
  items.value = [{
    ...restoredItem,
    id: items.value.some((item) => item.id === restoredItem.id) ? createId("item") : restoredItem.id,
    updatedAt: new Date().toISOString(),
  }, ...items.value];
  deletedItems.value = deletedItems.value.filter((item) => item.id !== id);
  recyclePopup.value = false;
  showToast("物品已恢复");
}

function purgeDeletedItem(id: string) {
  deletedItems.value = deletedItems.value.filter((item) => item.id !== id);
  showToast("已彻底删除");
}

function openImportExport() {
  settingsPopup.value = false;
  importInfo.value.dataStr = "";
  importInfo.value.fileName = "";
  importExportInfo.value.show = true;
}

function exportAllData() {
  const backupText = JSON.stringify({ items: items.value, deletedItems: deletedItems.value }, null, 2);
  const blob = new Blob([backupText], { type: "application/json;charset=utf-8" });
  const url = URL.createObjectURL(blob);
  const downloadLink = document.createElement("a");
  downloadLink.href = url;
  downloadLink.download = `warehouse-backup-${dayjs(new Date()).format(DATE_FORMAT)}.json`;
  document.body.appendChild(downloadLink);
  downloadLink.click();
  document.body.removeChild(downloadLink);
  URL.revokeObjectURL(url);
  settingsPopup.value = false;
  showToast("备份文件已导出");
}

function onImportFileLoaded(payload: { dataStr: string; fileName: string }) {
  importInfo.value.dataStr = payload.dataStr;
  importInfo.value.fileName = payload.fileName;
  showToast("备份文件已读取");
}

function importData() {
  try {
    const parsedData = JSON.parse(importInfo.value.dataStr) as WarehouseBackupData;
    if (!parsedData || typeof parsedData !== "object" || Array.isArray(parsedData)) {
      throw new Error("Invalid backup data");
    }
    const importedItems = normalizeItems(parsedData.items);
    const importedDeletedItems = normalizeDeletedItems(parsedData.deletedItems);
    if (!Array.isArray(parsedData.items) || importedItems.length !== parsedData.items.length) {
      throw new Error("Invalid items");
    }
    if (parsedData.deletedItems !== undefined && (
      !Array.isArray(parsedData.deletedItems)
      || importedDeletedItems.length !== parsedData.deletedItems.length
    )) {
      throw new Error("Invalid deleted items");
    }
    items.value = mergeItemsByName(items.value, importedItems);
    deletedItems.value = mergeItemsByName(
      deletedItems.value,
      purgeExpiredDeletedItems(importedDeletedItems),
    );
    importExportInfo.value.show = false;
    importInfo.value.dataStr = "";
    importInfo.value.fileName = "";
    showToast("总数据合并成功");
  } catch {
    showToast("导入失败，请检查备份文件");
  }
}

function resetAllData() {
  showConfirmDialog({
    title: "提示",
    message: "确认清除所有库房数据吗？",
    width: "250px",
  }).then(() => {
    items.value = [];
    deletedItems.value = [];
    itemStorage.remove();
    recycleStorage.remove();
    settingsPopup.value = false;
    showToast("数据已重置");
  }).catch(() => {});
}

function formatQuantity(quantity: number) {
  return Number.isInteger(quantity) ? String(quantity) : String(Number(quantity.toFixed(2)));
}

function scrollToTop() {
  window.scrollTo({ top: 0, behavior: "smooth" });
}

function blurSearchOnScroll() {
  if (document.activeElement === searchInput.value) searchInput.value?.blur();
}

function updateScrollTopVisibility() {
  showScrollTop.value = window.scrollY > SCROLL_TOP_THRESHOLD;
}

init();

onMounted(() => {
  updateScrollTopVisibility();
  window.addEventListener("scroll", updateScrollTopVisibility, { passive: true });
  window.addEventListener("touchmove", blurSearchOnScroll, { passive: true });
  window.addEventListener("wheel", blurSearchOnScroll, { passive: true });
});

onUnmounted(() => {
  window.removeEventListener("scroll", updateScrollTopVisibility);
  window.removeEventListener("touchmove", blurSearchOnScroll);
  window.removeEventListener("wheel", blurSearchOnScroll);
});
</script>

<style lang="scss" scoped>
.home-page {
  min-height: 100vh;
  padding: 0 20px calc(12px + env(safe-area-inset-bottom));
  background-color: color-mix(in srgb, var(--accent) 4%, var(--page-bg));
  color: var(--text-main);
}

.home-page,
.home-page * {
  box-sizing: border-box;
}

.warehouse-view {
  width: min(100%, 880px);
  margin: 0 auto;
}

.sticky-panel {
  --toolbar-inset: max(20px, calc((100% - 880px) / 2));
  position: sticky;
  top: 0;
  z-index: 20;
  margin: 0 -20px;
  padding-top: env(safe-area-inset-top);
  background-color: color-mix(in srgb, color-mix(in srgb, var(--accent) 4%, var(--page-bg)) 85%, transparent);
  -webkit-backdrop-filter: saturate(180%) blur(16px);
  backdrop-filter: saturate(180%) blur(16px);
  border-bottom: 1px solid color-mix(in srgb, var(--divider) 70%, transparent);
}

.page-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 18px var(--toolbar-inset) 12px;
}

.header-title {
  display: flex;
  align-items: baseline;
  gap: 10px;
  min-width: 0;
}

.page-header h1 {
  margin: 0;
  color: var(--text-strong);
  font-size: 28px;
  font-weight: 700;
  line-height: 1.15;
  letter-spacing: -0.02em;
  white-space: nowrap;
}

.item-count {
  padding: 3px 9px;
  border-radius: 999px;
  background: var(--accent-soft);
  color: var(--accent);
  font-size: 12px;
  font-weight: 600;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.header-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border: 1px solid var(--divider);
  border-radius: 12px;
  background: var(--surface);
  color: var(--text-main);
  font-size: 18px;
  transition: background-color 0.2s ease, color 0.2s ease, transform 0.15s var(--ease);
}

.header-icon:hover {
  color: var(--accent);
  background: var(--surface-soft);
}

.header-icon:active {
  transform: scale(0.94);
}

.search-section {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  gap: 10px;
  padding: 0 var(--toolbar-inset) 14px;
}

.search-box {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
  height: 46px;
  padding: 0 14px;
  border: 1px solid var(--divider);
  border-radius: var(--radius-md);
  color: var(--text-subtle);
  background: var(--surface);
  font-size: 17px;
  transition: border-color 0.2s ease, box-shadow 0.2s ease, color 0.2s ease;
}

.search-box:focus-within {
  border-color: var(--accent);
  color: var(--accent);
}

.search-box input {
  width: 100%;
  min-width: 0;
  border: 0;
  outline: 0;
  color: var(--text-main);
  background: transparent;
  font-size: 16px;
  font-weight: 400;
  -webkit-appearance: none;
  appearance: none;
}

.search-box input::placeholder {
  color: var(--text-subtle);
}

.search-box input::-webkit-search-cancel-button {
  cursor: pointer;
}

.create-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  height: 46px;
  padding: 0 18px;
  border: 0;
  border-radius: var(--radius-md);
  color: #ffffff;
  background: var(--accent);
  box-shadow: 0 4px 12px color-mix(in srgb, var(--accent) 24%, transparent);
  font-size: 15px;
  font-weight: 600;
  white-space: nowrap;
  transition: background-color 0.2s ease, transform 0.15s var(--ease);
}

.create-button:hover {
  background: var(--accent-strong);
}

.create-button:active {
  transform: scale(0.95);
}

.create-button .el-icon {
  transition: transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.create-button:hover .el-icon,
.create-button:active .el-icon {
  transform: rotate(90deg);
}

.header-icon .el-icon {
  transition: transform 0.5s var(--ease);
}

.header-icon:hover .el-icon {
  transform: rotate(45deg);
}

.waterfall-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 12px;
  padding: 18px 0 0;
}

.warehouse-card {
  position: relative;
  display: flex;
  flex-direction: column;
  min-width: 0;
  padding: 16px 16px 14px;
  border: 0;
  border-radius: 18px;
  background: linear-gradient(145deg, color-mix(in srgb, var(--accent) 4%, var(--surface)) 0%, color-mix(in srgb, var(--accent) 13%, var(--surface)) 100%);
  box-shadow:
    0 1px 2px rgba(17, 24, 39, 0.04),
    0 4px 10px rgba(17, 24, 39, 0.04),
    0 12px 28px color-mix(in srgb, var(--accent) 10%, transparent);
  cursor: pointer;
  overflow: hidden;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.warehouse-card:hover {
  box-shadow:
    0 1px 2px rgba(17, 24, 39, 0.05),
    0 6px 14px rgba(17, 24, 39, 0.05),
    0 16px 36px color-mix(in srgb, var(--accent) 14%, transparent);
}

.warehouse-card:active {
  background: linear-gradient(145deg, color-mix(in srgb, var(--accent) 8%, var(--surface)) 0%, color-mix(in srgb, var(--accent) 18%, var(--surface)) 100%);
}

.delete-btn {
  position: absolute;
  top: 12px;
  right: 12px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  padding: 0;
  border: 0;
  border-radius: 50%;
  color: var(--text-subtle);
  background: transparent;
  font-size: 16px;
  line-height: 1;
  transition: background-color 0.2s ease, color 0.2s ease;
}

.delete-btn:hover,
.delete-btn:active {
  color: var(--danger-text);
  background: var(--danger-bg);
}

.warehouse-card h2 {
  margin: 0;
  padding-right: 26px;
  color: var(--text-strong);
  font-size: 15px;
  font-weight: 600;
  line-height: 1.4;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.quantity-line {
  margin: 12px 0 0;
  color: var(--accent);
  font-size: 24px;
  font-weight: 650;
  line-height: 1.1;
  letter-spacing: -0.01em;
  font-variant-numeric: tabular-nums;
}

.quantity-unit {
  display: inline-block;
  margin-left: 6px;
  padding: 2px 7px;
  border-radius: 6px;
  color: var(--accent);
  background: color-mix(in srgb, var(--accent) 12%, var(--surface));
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0;
  vertical-align: 3px;
}

.updated-date {
  margin-top: 10px;
  color: var(--text-subtle);
  font-size: 11px;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.empty-state {
  display: grid;
  place-items: center;
  align-content: center;
  gap: 8px;
  min-height: 52vh;
  padding: 40px 16px;
  text-align: center;
}

.empty-state::before {
  content: "";
  width: 56px;
  height: 56px;
  margin-bottom: 10px;
  border-radius: 18px;
  background:
    linear-gradient(var(--accent), var(--accent)) center / 22px 2px no-repeat,
    linear-gradient(var(--accent), var(--accent)) center / 2px 22px no-repeat,
    var(--accent-soft);
  opacity: 0.9;
}

.empty-state.compact {
  min-height: 0;
  padding: 32px 12px 12px;
}

.empty-state.compact::before {
  width: 48px;
  height: 48px;
  background:
    linear-gradient(var(--text-subtle), var(--text-subtle)) center / 18px 2px no-repeat,
    var(--surface-muted);
}

.empty-state strong {
  color: var(--text-strong);
  font-size: 17px;
  font-weight: 600;
}

.empty-state span {
  color: var(--text-muted);
  font-size: 13px;
  line-height: 1.5;
}

.home-page :deep(.van-popup) {
  background: var(--surface);
  color: var(--text-main);
}

.home-page :deep(.edit-popup) {
  width: min(90vw, 380px);
  border-radius: 22px;
  background: var(--surface);
  box-shadow: var(--popup-shadow);
  overflow: hidden;
}

.simple-editor {
  padding: 24px 22px 20px;
}

.editor-title {
  display: grid;
  gap: 4px;
  margin-bottom: 18px;
}

.editor-title strong {
  color: var(--text-strong);
  font-size: 20px;
  font-weight: 650;
  letter-spacing: -0.01em;
}

.editor-title span {
  color: var(--text-muted);
  font-size: 13px;
}

.field-row {
  display: grid;
  gap: 8px;
  margin-top: 14px;
}

.field-row > span {
  color: var(--text-muted);
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.02em;
}

.simple-input {
  display: block;
  width: 100%;
  height: 44px;
  padding: 0 14px;
  border: 0;
  border-radius: 12px;
  outline: 0;
  color: var(--text-main);
  background: var(--field-bg);
  caret-color: var(--accent);
  font-size: 16px;
  font-weight: 500;
  transition: border-color 0.2s ease, background-color 0.2s ease, box-shadow 0.2s ease;
}

.simple-input::placeholder {
  color: var(--text-subtle);
  font-weight: 400;
}

.simple-input:focus {
  background: color-mix(in srgb, var(--accent) 6%, var(--surface));
  box-shadow: inset 0 0 0 1.5px color-mix(in srgb, var(--accent) 45%, transparent);
}

.quantity-stepper {
  display: grid;
  grid-template-columns: 44px minmax(0, 1fr) 44px;
  gap: 8px;
}

.stepper-btn {
  height: 44px;
  border: 1px solid var(--divider);
  border-radius: 12px;
  color: var(--accent);
  background: var(--surface);
  font-size: 22px;
  font-weight: 600;
  line-height: 1;
  transition: background-color 0.15s ease, border-color 0.15s ease, opacity 0.15s ease;
  touch-action: manipulation;
}

.stepper-btn:hover:not(:disabled) {
  border-color: var(--accent-border);
}

.stepper-btn:active:not(:disabled) {
  background: color-mix(in srgb, var(--accent) 10%, var(--surface));
}

.stepper-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.stepper-input {
  text-align: center;
  -moz-appearance: textfield;
  appearance: textfield;
}

.stepper-input::-webkit-outer-spin-button,
.stepper-input::-webkit-inner-spin-button {
  margin: 0;
  -webkit-appearance: none;
}

.unit-grid {
  display: grid;
  grid-template-columns: repeat(8, minmax(0, 1fr));
  gap: 6px;
}

.unit-option {
  height: 36px;
  border: 1px solid var(--divider);
  border-radius: 10px;
  color: var(--text-main);
  background: var(--surface);
  font-size: 14px;
  font-weight: 500;
  transition: background-color 0.15s ease, border-color 0.15s ease, color 0.15s ease;
}

.unit-option:hover {
  border-color: var(--accent-border);
}

.unit-option.active {
  border-color: var(--accent);
  color: #ffffff;
  background: var(--accent);
  font-weight: 600;
}

.simple-actions {
  display: grid;
  grid-template-columns: 1fr 1.4fr;
  gap: 10px;
  margin-top: 24px;
}

.simple-actions button {
  height: 44px;
  border: 0;
  border-radius: 12px;
  font-size: 15px;
  font-weight: 600;
  transition: background-color 0.2s ease, transform 0.15s var(--ease);
}

.simple-actions button:active {
  transform: scale(0.98);
}

.cancel-btn {
  color: var(--text-main);
  background: var(--surface-muted);
}

.confirm-btn {
  color: #ffffff;
  background: var(--submit-btn-bg);
  box-shadow: var(--submit-btn-shadow);
}

.confirm-btn:hover {
  background: var(--accent-strong);
}

.recycle-popup {
  max-height: 78vh;
  overflow-y: auto;
}

.recycle-list {
  display: grid;
  gap: 0;
  margin-top: 16px;
  border: 1px solid var(--divider);
  border-radius: var(--radius-md);
  overflow: hidden;
}

.recycle-item {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  gap: 12px;
  align-items: center;
  padding: 14px 16px;
  background: var(--surface);
}

.recycle-item + .recycle-item {
  border-top: 1px solid var(--divider);
}

.recycle-item strong,
.recycle-item span {
  display: block;
}

.recycle-item strong {
  color: var(--text-strong);
  font-size: 15px;
  font-weight: 600;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.recycle-item span {
  margin-top: 4px;
  color: var(--text-muted);
  font-size: 12px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.recycle-actions {
  display: flex;
  gap: 8px;
}

.recycle-actions button {
  height: 32px;
  padding: 0 14px;
  border: 0;
  border-radius: 999px;
  background: var(--accent-soft);
  color: var(--accent);
  font-size: 13px;
  font-weight: 600;
}

.recycle-actions button.danger {
  background: var(--danger-bg);
  color: var(--danger-text);
}

@media (max-width: 520px) {
  .home-page {
    padding-inline: 14px;
  }

  .sticky-panel {
    --toolbar-inset: 14px;
    margin: 0 -14px;
  }

  .page-header {
    padding-top: 14px;
  }

  .page-header h1 {
    font-size: 24px;
  }

  .create-button {
    padding: 0 14px;
  }

  .waterfall-list {
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 10px;
    padding-top: 14px;
  }

  .warehouse-card {
    padding: 14px 14px 12px;
    border-radius: 16px;
  }



  .delete-btn {
    top: 9px;
    right: 9px;
  }

  .quantity-line {
    margin-top: 10px;
    font-size: 22px;
  }

  .simple-editor {
    padding: 22px 18px 18px;
  }
}

</style>
