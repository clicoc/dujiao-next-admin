<script setup lang="ts">
import { onMounted, reactive, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import IdCell from '@/components/IdCell.vue'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Dialog, DialogScrollContent, DialogHeader, DialogTitle } from '@/components/ui/dialog'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { confirmAction } from '@/utils/confirm'
import { notifyError, notifySuccess } from '@/utils/notify'

const { t } = useI18n()
const loading = ref(true)
const mappings = ref<any[]>([])
const connections = ref<any[]>([])
const pagination = reactive({
  page: 1,
  page_size: 20,
  total: 0,
  total_page: 1,
})
const jumpPage = ref('')
const filters = reactive({
  connection_id: '__all__',
})
const syncingId = ref<number | null>(null)

// Import dialog
const showImportModal = ref(false)
const importForm = reactive({
  connection_id: '',
  upstream_product_id: '',
  category_id: '',
  slug: '',
})
const upstreamProducts = ref<any[]>([])
const loadingUpstream = ref(false)

const normalizeFilterValue = (value: string) => (value === '__all__' ? '' : value)

const fetchConnections = async () => {
  try {
    const res = await adminAPI.getSiteConnections({ page_size: 100 })
    connections.value = (res.data.data as any[]) || []
  } catch {
    connections.value = []
  }
}

const fetchMappings = async (page = 1) => {
  loading.value = true
  try {
    const connId = normalizeFilterValue(filters.connection_id)
    const res = await adminAPI.getProductMappings({
      page,
      page_size: pagination.page_size,
      connection_id: connId || undefined,
    })
    mappings.value = (res.data.data as any[]) || []
    const p = (res.data as any).pagination
    if (p) {
      pagination.page = p.page
      pagination.page_size = p.page_size
      pagination.total = p.total
      pagination.total_page = p.total_page
    }
  } catch {
    mappings.value = []
  } finally {
    loading.value = false
  }
}

const changePage = (page: number) => {
  if (page < 1 || page > pagination.total_page) return
  fetchMappings(page)
}

const jumpToPage = () => {
  if (!jumpPage.value) return
  const raw = Number(jumpPage.value)
  if (Number.isNaN(raw)) return
  const target = Math.min(Math.max(Math.floor(raw), 1), pagination.total_page)
  if (target === pagination.page) return
  changePage(target)
}

const handleFilterChange = () => {
  fetchMappings(1)
}

const openImportModal = () => {
  Object.assign(importForm, {
    connection_id: '',
    upstream_product_id: '',
    category_id: '',
    slug: '',
  })
  upstreamProducts.value = []
  showImportModal.value = true
}

const closeImportModal = () => {
  showImportModal.value = false
}

const fetchUpstreamProducts = async (connectionId: string) => {
  if (!connectionId) {
    upstreamProducts.value = []
    return
  }
  loadingUpstream.value = true
  try {
    const res = await adminAPI.getUpstreamProducts({
      connection_id: connectionId,
      page_size: 200,
    })
    upstreamProducts.value = (res.data.data as any[]) || []
  } catch {
    upstreamProducts.value = []
  } finally {
    loadingUpstream.value = false
  }
}

watch(
  () => importForm.connection_id,
  (value) => {
    importForm.upstream_product_id = ''
    fetchUpstreamProducts(value)
  }
)

const handleImport = async () => {
  try {
    await adminAPI.importUpstreamProduct({
      connection_id: Number(importForm.connection_id),
      upstream_product_id: Number(importForm.upstream_product_id),
      category_id: importForm.category_id ? Number(importForm.category_id) : undefined,
      slug: importForm.slug || undefined,
    })
    closeImportModal()
    fetchMappings(1)
    notifySuccess('导入成功')
  } catch (err: any) {
    notifyError(err?.response?.data?.message || err?.message)
  }
}

const handleSync = async (mapping: any) => {
  syncingId.value = mapping.id
  try {
    await adminAPI.syncProductMapping(mapping.id)
    notifySuccess('同步成功')
    fetchMappings(pagination.page)
  } catch (err: any) {
    notifyError('同步失败: ' + (err?.response?.data?.message || err?.message || ''))
  } finally {
    syncingId.value = null
  }
}

const handleToggleStatus = async (mapping: any) => {
  const nextActive = !mapping.is_active
  try {
    await adminAPI.updateProductMappingStatus(mapping.id, { is_active: nextActive })
    fetchMappings(pagination.page)
    notifySuccess()
  } catch (err: any) {
    notifyError(err?.response?.data?.message || err?.message)
  }
}

const handleDelete = async (mapping: any) => {
  const confirmed = await confirmAction({
    description: `确认删除映射 #${mapping.id}？`,
    confirmText: t('admin.common.delete'),
    variant: 'destructive',
  })
  if (!confirmed) return
  try {
    await adminAPI.deleteProductMapping(mapping.id)
    fetchMappings(pagination.page)
    notifySuccess()
  } catch (err: any) {
    notifyError(err?.response?.data?.message || err?.message)
  }
}

const getConnectionName = (connectionId: number) => {
  const conn = connections.value.find((c: any) => c.id === connectionId)
  return conn?.name || `#${connectionId}`
}

const formatTime = (raw?: string) => {
  if (!raw) return '-'
  const d = new Date(raw)
  if (Number.isNaN(d.getTime())) return '-'
  return d.toLocaleString()
}

onMounted(() => {
  fetchConnections()
  fetchMappings()
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-semibold">商品映射</h1>
      <Button @click="openImportModal">从上游导入</Button>
    </div>

    <div class="rounded-xl border border-border bg-card p-4 shadow-sm">
      <div class="flex flex-wrap items-center gap-3">
        <Select v-model="filters.connection_id" @update:modelValue="handleFilterChange">
          <SelectTrigger class="h-9 w-[220px]">
            <SelectValue placeholder="筛选连接" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="__all__">全部连接</SelectItem>
            <SelectItem v-for="conn in connections" :key="conn.id" :value="String(conn.id)">{{ conn.name }}</SelectItem>
          </SelectContent>
        </Select>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card">
      <Table>
        <TableHeader class="border-b border-border bg-muted/40 text-xs uppercase text-muted-foreground">
          <TableRow>
            <TableHead class="px-6 py-3">ID</TableHead>
            <TableHead class="px-6 py-3">连接</TableHead>
            <TableHead class="px-6 py-3">本地商品</TableHead>
            <TableHead class="px-6 py-3">上游商品 ID</TableHead>
            <TableHead class="px-6 py-3">状态</TableHead>
            <TableHead class="px-6 py-3">最后同步</TableHead>
            <TableHead class="px-6 py-3 text-right">操作</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody class="divide-y divide-border">
          <TableRow v-if="loading">
            <TableCell colspan="7" class="px-6 py-8 text-center text-muted-foreground">{{ t('admin.common.loading') }}</TableCell>
          </TableRow>
          <TableRow v-else-if="mappings.length === 0">
            <TableCell colspan="7" class="px-6 py-8 text-center text-muted-foreground">暂无数据</TableCell>
          </TableRow>
          <TableRow v-for="mapping in mappings" :key="mapping.id" class="hover:bg-muted/30">
            <TableCell class="px-6 py-4">
              <IdCell :value="mapping.id" />
            </TableCell>
            <TableCell class="px-6 py-4 text-sm text-foreground">{{ getConnectionName(mapping.connection_id) }}</TableCell>
            <TableCell class="px-6 py-4">
              <div v-if="mapping.local_product" class="text-sm">
                <span class="font-medium text-foreground">{{ mapping.local_product.name }}</span>
                <span class="ml-1 text-xs text-muted-foreground">#{{ mapping.local_product_id }}</span>
              </div>
              <span v-else class="text-xs text-muted-foreground">#{{ mapping.local_product_id || '-' }}</span>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs font-mono text-muted-foreground">{{ mapping.upstream_product_id }}</TableCell>
            <TableCell class="px-6 py-4">
              <span
                class="inline-flex rounded-full border px-2.5 py-1 text-xs"
                :class="mapping.is_active ? 'text-emerald-700 border-emerald-200 bg-emerald-50' : 'text-muted-foreground border-border bg-muted/30'"
              >
                {{ mapping.is_active ? '启用' : '禁用' }}
              </span>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ formatTime(mapping.last_synced_at) }}</TableCell>
            <TableCell class="px-6 py-4 text-right">
              <div class="flex items-center justify-end gap-2">
                <Button
                  size="sm"
                  variant="outline"
                  :disabled="syncingId === mapping.id"
                  @click="handleSync(mapping)"
                >
                  {{ syncingId === mapping.id ? '同步中...' : '同步' }}
                </Button>
                <Button size="sm" variant="outline" @click="handleToggleStatus(mapping)">
                  {{ mapping.is_active ? '禁用' : '启用' }}
                </Button>
                <Button size="sm" variant="destructive" @click="handleDelete(mapping)">{{ t('admin.common.delete') }}</Button>
              </div>
            </TableCell>
          </TableRow>
        </TableBody>
      </Table>

      <div v-if="pagination.total_page > 1" class="flex flex-wrap items-center justify-between gap-3 border-t border-border px-6 py-4">
        <span class="text-xs text-muted-foreground">
          {{ t('admin.common.pageInfo', { total: pagination.total, page: pagination.page, totalPage: pagination.total_page }) }}
        </span>
        <div class="flex flex-wrap items-center gap-2">
          <Input v-model="jumpPage" type="number" min="1" :max="pagination.total_page" class="h-8 w-20" :placeholder="t('admin.common.jumpPlaceholder')" />
          <Button variant="outline" size="sm" class="h-8" @click="jumpToPage">{{ t('admin.common.jumpTo') }}</Button>
          <Button variant="outline" size="sm" class="h-8" :disabled="pagination.page <= 1" @click="changePage(pagination.page - 1)">
            {{ t('admin.common.prevPage') }}
          </Button>
          <Button variant="outline" size="sm" class="h-8" :disabled="pagination.page >= pagination.total_page" @click="changePage(pagination.page + 1)">
            {{ t('admin.common.nextPage') }}
          </Button>
        </div>
      </div>
    </div>

    <Dialog v-model:open="showImportModal" @update:open="(value: boolean) => { if (!value) closeImportModal() }">
      <DialogScrollContent class="w-full max-w-xl" @interact-outside="(e: Event) => e.preventDefault()">
        <DialogHeader>
          <DialogTitle>从上游导入商品</DialogTitle>
        </DialogHeader>

        <form class="space-y-6" @submit.prevent="handleImport">
          <div class="space-y-4">
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">选择连接</label>
              <Select v-model="importForm.connection_id">
                <SelectTrigger class="h-9 w-full">
                  <SelectValue placeholder="选择连接" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem v-for="conn in connections" :key="conn.id" :value="String(conn.id)">{{ conn.name }}</SelectItem>
                </SelectContent>
              </Select>
            </div>

            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">上游商品</label>
              <Select v-model="importForm.upstream_product_id" :disabled="!importForm.connection_id || loadingUpstream">
                <SelectTrigger class="h-9 w-full">
                  <SelectValue :placeholder="loadingUpstream ? '加载中...' : '选择上游商品'" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem v-for="product in upstreamProducts" :key="product.id" :value="String(product.id)">
                    {{ product.name }} (#{{ product.id }})
                  </SelectItem>
                </SelectContent>
              </Select>
            </div>

            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">本地分类 ID（可选）</label>
              <Input v-model="importForm.category_id" type="number" placeholder="分类 ID" />
            </div>

            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">Slug（可选）</label>
              <Input v-model="importForm.slug" placeholder="product-slug" />
            </div>
          </div>

          <div class="flex justify-end gap-3 border-t border-border pt-6">
            <Button type="button" variant="outline" @click="closeImportModal">{{ t('admin.common.cancel') }}</Button>
            <Button type="submit" :disabled="!importForm.connection_id || !importForm.upstream_product_id">导入</Button>
          </div>
        </form>
      </DialogScrollContent>
    </Dialog>
  </div>
</template>
