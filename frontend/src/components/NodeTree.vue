<template>
  <div class="node-tree-container">
    <div class="tree-header">
      <h3 class="text-lg font-bold text-cyan-400">OPC-UA 节点树</h3>
      <el-tag :type="store.isConnected ? 'success' : 'danger'" size="small">
        {{ store.isConnected ? '已连接' : '未连接' }}
      </el-tag>
    </div>

    <div ref="searchSectionRef" class="search-section">
      <el-input
        v-model="searchKeyword"
        placeholder="搜索节点名称或编号..."
        clearable
        size="default"
        class="search-input"
        @input="handleSearchInput"
        @focus="handleSearchFocus"
        @keyup.enter="handleSearchEnter"
      >
        <template #prefix>
          <el-icon><Search /></el-icon>
        </template>
      </el-input>
    </div>

    <Teleport to="body">
      <div
        v-if="showSearchResult && searchResults.length > 0"
        class="search-results search-results-dropdown"
        :style="dropdownStyle"
      >
        <div
          v-for="(node, index) in searchResults"
          :key="node.id"
          class="search-result-item"
          :class="{ active: hoveredIndex === index }"
          @click="handleSelectSearchResult(node)"
          @mouseenter="hoveredIndex = index"
        >
          <div class="result-item-left">
            <el-icon v-if="node.type === 'Object'" class="text-yellow-400">
              <Folder />
            </el-icon>
            <el-icon v-else-if="node.type === 'Variable'" class="text-green-400">
              <DataLine />
            </el-icon>
            <span class="result-name">{{ highlightMatch(node.name) }}</span>
          </div>
          <div class="result-item-right">
            <span class="result-nodeid">{{ highlightMatch(node.nodeId) }}</span>
          </div>
        </div>
      </div>

      <div
        v-if="showSearchResult && searchKeyword.trim() && searchResults.length === 0"
        class="search-results search-no-result search-results-dropdown"
        :style="dropdownStyle"
      >
        <el-empty description="未找到匹配节点" :image-size="40" />
      </div>
    </Teleport>

    <el-tree
      ref="treeRef"
      :data="store.nodeTree"
      :props="treeProps"
      node-key="id"
      highlight-current
      default-expand-all
      @node-click="handleNodeClick"
      class="dark-tree"
    >
      <template #default="{ node, data }">
        <span class="custom-tree-node">
          <el-icon v-if="data.type === 'Object'" class="text-yellow-400">
            <Folder />
          </el-icon>
          <el-icon v-else-if="data.type === 'Variable'" class="text-green-400">
            <DataLine />
          </el-icon>
          <span class="node-label" :class="{ 'search-match': isSearchMatch(data) }">
            {{ data.name }}
          </span>
          <el-tag
            v-if="data.type === 'Variable' && data.quality"
            :type="data.quality === 'Good' ? 'success' : data.quality === 'Bad' ? 'danger' : 'warning'"
            size="small"
            class="ml-2"
          >
            {{ data.quality }}
          </el-tag>
          <span v-if="data.type === 'Variable' && data.value !== undefined" class="node-value">
            {{ data.value }}{{ data.unit ? ' ' + data.unit : '' }}
          </span>
        </span>
      </template>
    </el-tree>

    <div v-if="store.selectedNode" class="node-detail-panel">
      <el-divider />
      <h4 class="text-sm font-bold text-cyan-300 mb-2">节点详情</h4>
      <el-descriptions :column="1" size="small" border class="dark-descriptions">
        <el-descriptions-item label="名称">{{ store.selectedNode.name }}</el-descriptions-item>
        <el-descriptions-item label="Node ID">{{ store.selectedNode.nodeId }}</el-descriptions-item>
        <el-descriptions-item label="类型">
          <el-tag size="small">{{ store.selectedNode.type }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.dataType" label="数据类型">
          {{ store.selectedNode.dataType }}
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.value !== undefined" label="当前值">
          <span class="text-green-400 font-mono">
            {{ store.selectedNode.value }}{{ store.selectedNode.unit ? ' ' + store.selectedNode.unit : '' }}
          </span>
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.quality" label="质量码">
          <el-tag
            :type="store.selectedNode.quality === 'Good' ? 'success' : 'danger'"
            size="small"
          >
            {{ store.selectedNode.quality }}
          </el-tag>
        </el-descriptions-item>
        <el-descriptions-item v-if="store.selectedNode.description" label="描述">
          {{ store.selectedNode.description }}
        </el-descriptions-item>
      </el-descriptions>

      <div class="mt-3 flex gap-2">
        <el-button
          v-if="store.selectedNode.type === 'Variable'"
          type="primary"
          size="small"
          @click="handleSubscribe"
        >
          {{ isSubscribed ? '取消订阅' : '订阅' }}
        </el-button>
        <el-button
          v-if="store.selectedNode.type === 'Variable'"
          type="info"
          size="small"
          @click="handleReadValue"
        >
          读取
        </el-button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import { Folder, DataLine, Search } from '@element-plus/icons-vue'
import { ElMessage, ElTree } from 'element-plus'
import { useOpcuaStore } from '../store/opcua'
import type { OPCUANode } from '../types'

const store = useOpcuaStore()

const treeRef = ref<InstanceType<typeof ElTree> | null>(null)
const searchSectionRef = ref<HTMLElement | null>(null)
const searchKeyword = ref('')
const searchResults = ref<OPCUANode[]>([])
const showSearchResult = ref(false)
const hoveredIndex = ref(-1)
const dropdownStyle = reactive({ left: '0px', top: '0px', width: '0px' })

const treeProps = {
  children: 'children',
  label: 'name'
}

const isSubscribed = computed(() => {
  if (!store.selectedNode) return false
  return store.subscriptions.has(store.selectedNode.id)
})

function updateDropdownPosition() {
  if (!searchSectionRef.value) return
  const rect = searchSectionRef.value.getBoundingClientRect()
  dropdownStyle.left = `${rect.left}px`
  dropdownStyle.top = `${rect.bottom + 4}px`
  dropdownStyle.width = `${rect.width}px`
}

function handleSearchInput() {
  doSearch()
  showSearchResult.value = true
  hoveredIndex.value = -1
  updateDropdownPosition()
}

function handleSearchFocus() {
  if (searchKeyword.value.trim()) {
    doSearch()
  }
  showSearchResult.value = true
  nextTick(updateDropdownPosition)
}

function handleSearchEnter() {
  if (hoveredIndex.value >= 0 && hoveredIndex.value < searchResults.value.length) {
    handleSelectSearchResult(searchResults.value[hoveredIndex.value])
  } else if (searchResults.value.length > 0) {
    handleSelectSearchResult(searchResults.value[0])
  }
}

function doSearch() {
  const keyword = searchKeyword.value.trim()
  searchResults.value = keyword ? store.searchNodes(keyword).slice(0, 20) : []
}

function highlightMatch(text: string): string {
  return text
}

function isSearchMatch(node: OPCUANode): boolean {
  if (!searchKeyword.value.trim()) return false
  const kw = searchKeyword.value.toLowerCase().trim()
  return (
    node.name.toLowerCase().includes(kw) ||
    node.nodeId.toLowerCase().includes(kw) ||
    node.id.toLowerCase().includes(kw)
  )
}

function handleSelectSearchResult(node: OPCUANode) {
  showSearchResult.value = false
  searchKeyword.value = ''
  locateAndSelectNode(node.id)
}

async function locateAndSelectNode(nodeId: string) {
  const path = store.getNodePath(nodeId)
  if (path.length === 0) return

  await nextTick()

  const parentKeys = path.slice(0, -1)
  for (const key of parentKeys) {
    const node: any = treeRef.value?.getNode(key)
    if (node) {
      node.expanded = true
    }
  }

  await nextTick()

  treeRef.value?.setCurrentKey(nodeId)

  await nextTick()

  const targetNode: any = treeRef.value?.getNode(nodeId)
  if (targetNode && targetNode.$el) {
    targetNode.$el.scrollIntoView({ behavior: 'smooth', block: 'center' })
  }

  const nodeData = store.findNodeById(nodeId)
  if (nodeData) {
    store.selectNode(nodeData)
  }
}

function handleNodeClick(data: OPCUANode) {
  store.selectNode(data)
}

function handleSubscribe() {
  if (!store.selectedNode) return
  if (isSubscribed.value) {
    store.removeSubscription(store.selectedNode.id)
    ElMessage.success(`已取消订阅: ${store.selectedNode.name}`)
  } else {
    store.addSubscription(store.selectedNode.id)
    ElMessage.success(`已订阅: ${store.selectedNode.name}`)
  }
}

function handleReadValue() {
  if (!store.selectedNode) return
  ElMessage.success(`${store.selectedNode.name} = ${store.selectedNode.value} ${store.selectedNode.unit || ''}`)
}

function handleClickOutside(e: MouseEvent) {
  const target = e.target as HTMLElement
  const inSearchSection = searchSectionRef.value?.contains(target)
  const inDropdown = target.closest('.search-results-dropdown')
  if (!inSearchSection && !inDropdown) {
    showSearchResult.value = false
  }
}

function handleScrollOrResize() {
  if (showSearchResult.value) {
    updateDropdownPosition()
  }
}

watch(showSearchResult, (val) => {
  if (val) {
    nextTick(updateDropdownPosition)
  }
})

onMounted(() => {
  document.addEventListener('click', handleClickOutside, true)
  window.addEventListener('resize', handleScrollOrResize)
  window.addEventListener('scroll', handleScrollOrResize, true)
})

onBeforeUnmount(() => {
  document.removeEventListener('click', handleClickOutside, true)
  window.removeEventListener('resize', handleScrollOrResize)
  window.removeEventListener('scroll', handleScrollOrResize, true)
})
</script>

<style scoped>
.node-tree-container {
  height: 100%;
  overflow-y: auto;
  padding: 12px;
}

.tree-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.search-section {
  margin-bottom: 12px;
}

.search-input {
  width: 100%;
}

.search-results {
  max-height: 320px;
  overflow-y: auto;
  background: #1e293b;
  border: 1px solid rgba(71, 85, 105, 0.8);
  border-radius: 6px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
}

.search-results-dropdown {
  position: fixed;
  z-index: 9999;
}

.search-no-result {
  padding: 16px 0;
}

.search-result-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 12px;
  cursor: pointer;
  border-bottom: 1px solid rgba(71, 85, 105, 0.3);
  transition: background-color 0.15s;
}

.search-result-item:last-child {
  border-bottom: none;
}

.search-result-item:hover,
.search-result-item.active {
  background: rgba(6, 182, 212, 0.15);
}

.result-item-left {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
  min-width: 0;
}

.result-name {
  font-size: 13px;
  color: #e0e0e0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.result-item-right {
  flex-shrink: 0;
  margin-left: 12px;
}

.result-nodeid {
  font-size: 11px;
  color: #64748b;
  font-family: monospace;
  white-space: nowrap;
}

.custom-tree-node {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 13px;
  flex: 1;
  overflow: hidden;
}

.node-label {
  white-space: nowrap;
}

.node-label.search-match {
  color: #fbbf24;
  font-weight: 500;
}

.node-value {
  margin-left: auto;
  font-family: monospace;
  font-size: 12px;
  color: #67c23a;
  padding-left: 8px;
}

.node-detail-panel {
  padding: 8px 0;
}

:deep(.el-tree) {
  background: transparent !important;
  color: #e0e0e0 !important;
}

:deep(.el-tree-node__content:hover) {
  background: rgba(6, 182, 212, 0.1) !important;
}

:deep(.el-tree-node.is-current > .el-tree-node__content) {
  background: rgba(6, 182, 212, 0.2) !important;
}

:deep(.el-descriptions) {
  --el-descriptions-item-bordered-label-background: #1f2937;
}

:deep(.el-input__wrapper) {
  background: rgba(30, 41, 59, 0.8);
  box-shadow: 0 0 0 1px rgba(71, 85, 105, 0.5) inset;
}

:deep(.el-input__wrapper:hover) {
  box-shadow: 0 0 0 1px rgba(6, 182, 212, 0.5) inset;
}

:deep(.el-input__wrapper.is-focus) {
  box-shadow: 0 0 0 1px rgba(6, 182, 212, 0.8) inset;
}
</style>
