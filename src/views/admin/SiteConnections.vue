<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
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
const connections = ref<any[]>([])
const pagination = reactive({
  page: 1,
  page_size: 20,
  total: 0,
  total_page: 1,
})
const jumpPage = ref('')
const showModal = ref(false)
const isEditing = ref(false)
const editingId = ref<number | null>(null)
const pingingId = ref<number | null>(null)

const form = reactive({
  name: '',
  base_url: '',
  api_key: '',
  api_secret: '',
  protocol: 'dujiao-next',
  callback_url: '',
  retry_max: 3,
  retry_intervals: '30,60,120',
})

const fetchConnections = async (page = 1) => {
  loading.value = true
  try {
    const res = await adminAPI.getSiteConnections({
      page,
      page_size: pagination.page_size,
    })
    connections.value = (res.data.data as any[]) || []
    const p = (res.data as any).pagination
    if (p) {
      pagination.page = p.page
      pagination.page_size = p.page_size
      pagination.total = p.total
      pagination.total_page = p.total_page
    }
  } catch {
    connections.value = []
  } finally {
    loading.value = false
  }
}

const changePage = (page: number) => {
  if (page < 1 || page > pagination.total_page) return
  fetchConnections(page)
}

const jumpToPage = () => {
  if (!jumpPage.value) return
  const raw = Number(jumpPage.value)
  if (Number.isNaN(raw)) return
  const target = Math.min(Math.max(Math.floor(raw), 1), pagination.total_page)
  if (target === pagination.page) return
  changePage(target)
}

const resetForm = () => {
  Object.assign(form, {
    name: '',
    base_url: '',
    api_key: '',
    api_secret: '',
    protocol: 'dujiao-next',
    callback_url: '',
    retry_max: 3,
    retry_intervals: '30,60,120',
  })
}

const openCreateModal = () => {
  isEditing.value = false
  editingId.value = null
  resetForm()
  showModal.value = true
}

const openEditModal = (conn: any) => {
  isEditing.value = true
  editingId.value = conn.id
  Object.assign(form, {
    name: conn.name || '',
    base_url: conn.base_url || '',
    api_key: conn.api_key || '',
    api_secret: conn.api_secret || '',
    protocol: conn.protocol || 'dujiao-next',
    callback_url: conn.callback_url || '',
    retry_max: conn.retry_max ?? 3,
    retry_intervals: Array.isArray(conn.retry_intervals)
      ? conn.retry_intervals.join(',')
      : conn.retry_intervals || '30,60,120',
  })
  showModal.value = true
}

const closeModal = () => {
  showModal.value = false
}

const buildPayload = () => {
  const intervals = form.retry_intervals
    .split(',')
    .map((s: string) => Number(s.trim()))
    .filter((n: number) => !Number.isNaN(n) && n > 0)
  return {
    name: form.name,
    base_url: form.base_url,
    api_key: form.api_key,
    api_secret: form.api_secret,
    protocol: form.protocol,
    callback_url: form.callback_url,
    retry_max: Number(form.retry_max) || 3,
    retry_intervals: intervals,
  }
}

const handleSubmit = async () => {
  try {
    const payload = buildPayload()
    if (isEditing.value && editingId.value) {
      await adminAPI.updateSiteConnection(editingId.value, payload)
    } else {
      await adminAPI.createSiteConnection(payload)
    }
    closeModal()
    fetchConnections(isEditing.value ? pagination.page : 1)
    notifySuccess()
  } catch (err: any) {
    notifyError(err?.response?.data?.message || err?.message)
  }
}

const handlePing = async (conn: any) => {
  pingingId.value = conn.id
  try {
    await adminAPI.pingSiteConnection(conn.id)
    notifySuccess('Ping 成功')
    fetchConnections(pagination.page)
  } catch (err: any) {
    notifyError('Ping 失败: ' + (err?.response?.data?.message || err?.message || ''))
  } finally {
    pingingId.value = null
  }
}

const handleToggleStatus = async (conn: any) => {
  const nextStatus = conn.status === 'active' ? 'disabled' : 'active'
  try {
    await adminAPI.updateSiteConnectionStatus(conn.id, { status: nextStatus })
    fetchConnections(pagination.page)
    notifySuccess()
  } catch (err: any) {
    notifyError(err?.response?.data?.message || err?.message)
  }
}

const handleDelete = async (conn: any) => {
  const confirmed = await confirmAction({
    description: `确认删除连接「${conn.name || '#' + conn.id}」？`,
    confirmText: t('admin.common.delete'),
    variant: 'destructive',
  })
  if (!confirmed) return
  try {
    await adminAPI.deleteSiteConnection(conn.id)
    fetchConnections(pagination.page)
    notifySuccess()
  } catch (err: any) {
    notifyError(err?.response?.data?.message || err?.message)
  }
}

const statusBadgeClass = (status: string) => {
  switch (status) {
    case 'active':
      return 'text-emerald-700 border-emerald-200 bg-emerald-50'
    case 'pending':
      return 'text-yellow-700 border-yellow-200 bg-yellow-50'
    default:
      return 'text-muted-foreground border-border bg-muted/30'
  }
}

const statusLabel = (status: string) => {
  switch (status) {
    case 'active':
      return '已激活'
    case 'pending':
      return '待激活'
    case 'disabled':
      return '已禁用'
    default:
      return status
  }
}

const formatTime = (raw?: string) => {
  if (!raw) return '-'
  const d = new Date(raw)
  if (Number.isNaN(d.getTime())) return '-'
  return d.toLocaleString()
}

onMounted(() => {
  fetchConnections()
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-semibold">连接管理</h1>
      <Button @click="openCreateModal">新建连接</Button>
    </div>

    <div class="rounded-xl border border-border bg-card">
      <Table>
        <TableHeader class="border-b border-border bg-muted/40 text-xs uppercase text-muted-foreground">
          <TableRow>
            <TableHead class="px-6 py-3">ID</TableHead>
            <TableHead class="px-6 py-3">名称</TableHead>
            <TableHead class="px-6 py-3">Base URL</TableHead>
            <TableHead class="px-6 py-3">协议</TableHead>
            <TableHead class="px-6 py-3">状态</TableHead>
            <TableHead class="px-6 py-3">最后 Ping</TableHead>
            <TableHead class="px-6 py-3 text-right">操作</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody class="divide-y divide-border">
          <TableRow v-if="loading">
            <TableCell colspan="7" class="px-6 py-8 text-center text-muted-foreground">{{ t('admin.common.loading') }}</TableCell>
          </TableRow>
          <TableRow v-else-if="connections.length === 0">
            <TableCell colspan="7" class="px-6 py-8 text-center text-muted-foreground">暂无数据</TableCell>
          </TableRow>
          <TableRow v-for="conn in connections" :key="conn.id" class="hover:bg-muted/30">
            <TableCell class="px-6 py-4">
              <IdCell :value="conn.id" />
            </TableCell>
            <TableCell class="px-6 py-4 font-medium text-foreground">{{ conn.name }}</TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground font-mono">{{ conn.base_url }}</TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ conn.protocol }}</TableCell>
            <TableCell class="px-6 py-4">
              <span
                class="inline-flex rounded-full border px-2.5 py-1 text-xs"
                :class="statusBadgeClass(conn.status)"
              >
                {{ statusLabel(conn.status) }}
              </span>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ formatTime(conn.last_ping_at) }}</TableCell>
            <TableCell class="px-6 py-4 text-right">
              <div class="flex items-center justify-end gap-2">
                <Button size="sm" variant="outline" @click="openEditModal(conn)">{{ t('admin.common.edit') }}</Button>
                <Button
                  size="sm"
                  variant="outline"
                  :disabled="pingingId === conn.id"
                  @click="handlePing(conn)"
                >
                  {{ pingingId === conn.id ? 'Ping...' : 'Ping' }}
                </Button>
                <Button size="sm" variant="outline" @click="handleToggleStatus(conn)">
                  {{ conn.status === 'active' ? '禁用' : '启用' }}
                </Button>
                <Button size="sm" variant="destructive" @click="handleDelete(conn)">{{ t('admin.common.delete') }}</Button>
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

    <Dialog v-model:open="showModal" @update:open="(value: boolean) => { if (!value) closeModal() }">
      <DialogScrollContent class="w-full max-w-2xl" @interact-outside="(e: Event) => e.preventDefault()">
        <DialogHeader>
          <DialogTitle>{{ isEditing ? '编辑连接' : '新建连接' }}</DialogTitle>
        </DialogHeader>

        <form class="space-y-6" @submit.prevent="handleSubmit">
          <div class="grid grid-cols-1 gap-4 md:grid-cols-2">
            <div class="md:col-span-2">
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">名称</label>
              <Input v-model="form.name" required placeholder="连接名称" />
            </div>
            <div class="md:col-span-2">
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">Base URL</label>
              <Input v-model="form.base_url" required placeholder="https://example.com" />
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">API Key</label>
              <Input v-model="form.api_key" placeholder="API Key" />
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">API Secret</label>
              <Input v-model="form.api_secret" type="password" placeholder="API Secret" />
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">协议</label>
              <Select v-model="form.protocol">
                <SelectTrigger class="h-9 w-full">
                  <SelectValue placeholder="选择协议" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="dujiao-next">dujiao-next</SelectItem>
                </SelectContent>
              </Select>
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">回调 URL</label>
              <Input v-model="form.callback_url" placeholder="https://your-site.com/callback" />
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">最大重试次数</label>
              <Input v-model.number="form.retry_max" type="number" min="0" placeholder="3" />
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">重试间隔（秒，逗号分隔）</label>
              <Input v-model="form.retry_intervals" placeholder="30,60,120" />
            </div>
          </div>

          <div class="flex justify-end gap-3 border-t border-border pt-6">
            <Button type="button" variant="outline" @click="closeModal">{{ t('admin.common.cancel') }}</Button>
            <Button type="submit">{{ isEditing ? t('admin.common.save') : '创建' }}</Button>
          </div>
        </form>
      </DialogScrollContent>
    </Dialog>
  </div>
</template>
