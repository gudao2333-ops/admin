<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import { useDebounceFn } from '@vueuse/core'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import type { AdminSubsite } from '@/api/types'
import IdCell from '@/components/IdCell.vue'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import TableSkeleton from '@/components/TableSkeleton.vue'
import { confirmAction } from '@/utils/confirm'
import { formatDate } from '@/utils/format'
import { notifyError, notifySuccess } from '@/utils/notify'

const { t } = useI18n()
const loading = ref(true)
const operating = ref(false)
const rows = ref<AdminSubsite[]>([])

const filters = reactive({
  keyword: '',
  status: '__all__',
})

const pagination = ref({
  page: 1,
  page_size: 20,
  total: 0,
  total_page: 1,
})

const normalizeFilterValue = (value: string) => (value === '__all__' ? '' : value)

const fetchRows = async (page = 1) => {
  loading.value = true
  try {
    const response = await adminAPI.getSubsites({
      page,
      page_size: pagination.value.page_size,
      keyword: filters.keyword || undefined,
      status: normalizeFilterValue(filters.status) || undefined,
    })
    rows.value = response.data?.data || []
    pagination.value = response.data?.pagination || pagination.value
  } catch {
    rows.value = []
  } finally {
    loading.value = false
  }
}

const handleSearch = () => fetchRows(1)
const debouncedSearch = useDebounceFn(handleSearch, 300)

const changePage = (page: number) => {
  if (page < 1 || page > pagination.value.total_page) return
  fetchRows(page)
}

const statusClass = (status?: string) => {
  if (status === 'active') return 'border-emerald-200 bg-emerald-50 text-emerald-700'
  if (status === 'pending') return 'border-amber-200 bg-amber-50 text-amber-700'
  if (status === 'suspended') return 'border-zinc-200 bg-zinc-50 text-zinc-700'
  return 'border-border bg-muted/30 text-muted-foreground'
}

const statusLabel = (status?: string) => {
  if (!status) return '-'
  return t(`admin.subsites.list.status.${status}`, status)
}

const updateStatus = async (row: AdminSubsite, status: string) => {
  const confirmed = await confirmAction({ description: t('admin.subsites.list.changeStatusConfirm', { id: row.id, status }) })
  if (!confirmed) return

  operating.value = true
  try {
    await adminAPI.updateSubsiteStatus(row.id, { status })
    notifySuccess(t('admin.subsites.list.changeStatusSuccess'))
    await fetchRows(pagination.value.page)
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.list.changeStatusFailed'))
  } finally {
    operating.value = false
  }
}

onMounted(() => {
  fetchRows()
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-semibold">{{ t('admin.subsites.list.title') }}</h1>
    </div>

    <div class="rounded-xl border border-border bg-card p-4 shadow-sm">
      <div class="flex flex-wrap items-center gap-3">
        <div class="w-full md:w-56">
          <Input v-model="filters.keyword" :placeholder="t('admin.subsites.list.filters.keyword')" @update:modelValue="debouncedSearch" />
        </div>
        <div class="w-full md:w-44">
          <Select v-model="filters.status" @update:modelValue="handleSearch">
            <SelectTrigger class="h-9 w-full">
              <SelectValue :placeholder="t('admin.subsites.list.filters.allStatus')" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">{{ t('admin.subsites.list.filters.allStatus') }}</SelectItem>
              <SelectItem value="pending">{{ t('admin.subsites.list.status.pending') }}</SelectItem>
              <SelectItem value="active">{{ t('admin.subsites.list.status.active') }}</SelectItem>
              <SelectItem value="suspended">{{ t('admin.subsites.list.status.suspended') }}</SelectItem>
              <SelectItem value="closed">{{ t('admin.subsites.list.status.closed') }}</SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="flex-1"></div>
        <Button size="sm" variant="outline" @click="fetchRows(pagination.page)">{{ t('admin.common.refresh') }}</Button>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card">
      <Table>
        <TableHeader class="border-b border-border bg-muted/40 text-xs uppercase text-muted-foreground">
          <TableRow>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.list.table.id') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.list.table.owner') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.list.table.site') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.list.table.price') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.list.table.profit') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.list.table.status') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.list.table.createdAt') }}</TableHead>
            <TableHead class="px-6 py-3 text-right">{{ t('admin.subsites.list.table.action') }}</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody class="divide-y divide-border">
          <TableRow v-if="loading">
            <TableCell :colspan="8" class="p-0"><TableSkeleton :columns="8" :rows="5" /></TableCell>
          </TableRow>
          <TableRow v-else-if="rows.length === 0">
            <TableCell colspan="8" class="px-6 py-8 text-center text-muted-foreground">{{ t('admin.subsites.list.empty') }}</TableCell>
          </TableRow>
          <TableRow v-for="item in rows" :key="item.id">
            <TableCell class="px-6 py-4"><IdCell :value="item.id" /></TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">
              <div class="text-foreground">{{ item.user?.display_name || '-' }}</div>
              <div>{{ item.user?.email || '-' }}</div>
              <div class="font-mono">#{{ item.user_id }}</div>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs">
              <div class="font-medium">{{ item.site_name || '-' }}</div>
              <div class="font-mono text-muted-foreground">{{ item.domain || '-' }}</div>
            </TableCell>
            <TableCell class="px-6 py-4 font-mono text-xs">{{ item.opening_price || '0.00' }}</TableCell>
            <TableCell class="px-6 py-4 font-mono text-xs text-muted-foreground">
              <div>{{ item.total_profit || '0.00' }}</div>
              <div>{{ t('admin.subsites.list.withdrawableProfit') }}: {{ item.withdrawable_profit || '0.00' }}</div>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs">
              <span class="inline-flex rounded-full border px-2.5 py-1" :class="statusClass(item.status)">{{ statusLabel(item.status) }}</span>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ formatDate(item.created_at) }}</TableCell>
            <TableCell class="px-6 py-4 text-right">
              <div class="flex justify-end gap-2">
                <Button size="sm" variant="outline" :disabled="operating || item.status === 'active'" @click="updateStatus(item, 'active')">{{ t('admin.subsites.list.actions.activate') }}</Button>
                <Button size="sm" variant="outline" :disabled="operating || item.status === 'suspended'" @click="updateStatus(item, 'suspended')">{{ t('admin.subsites.list.actions.suspend') }}</Button>
              </div>
            </TableCell>
          </TableRow>
        </TableBody>
      </Table>

      <div v-if="pagination.total_page > 1" class="flex flex-wrap items-center justify-end gap-2 border-t border-border px-6 py-4">
        <Button variant="outline" size="sm" class="h-8" :disabled="pagination.page <= 1" @click="changePage(pagination.page - 1)">
          {{ t('admin.common.prevPage') }}
        </Button>
        <Button variant="outline" size="sm" class="h-8" :disabled="pagination.page >= pagination.total_page" @click="changePage(pagination.page + 1)">
          {{ t('admin.common.nextPage') }}
        </Button>
      </div>
    </div>
  </div>
</template>
