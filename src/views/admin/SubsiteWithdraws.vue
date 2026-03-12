<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import { useDebounceFn } from '@vueuse/core'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import type { AdminSubsiteWithdraw } from '@/api/types'
import IdCell from '@/components/IdCell.vue'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import TableSkeleton from '@/components/TableSkeleton.vue'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { confirmAction } from '@/utils/confirm'
import { notifyError, notifySuccess } from '@/utils/notify'
import { formatDate } from '@/utils/format'

const { t } = useI18n()
const loading = ref(true)
const operating = ref(false)
const rows = ref<AdminSubsiteWithdraw[]>([])

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
    const response = await adminAPI.getSubsiteWithdraws({
      page,
      page_size: pagination.value.page_size,
      keyword: filters.keyword || undefined,
      status: normalizeFilterValue(filters.status) || undefined,
    })
    rows.value = response.data.data || []
    pagination.value = response.data.pagination || pagination.value
  } catch {
    rows.value = []
  } finally {
    loading.value = false
  }
}

const handleSearch = () => fetchRows(1)
const debouncedSearch = useDebounceFn(handleSearch, 300)

const statusLabel = (status?: string) => {
  if (!status) return '-'
  return t(`admin.subsites.withdraws.status.${status}`, status)
}

const statusClass = (status?: string) => {
  if (status === 'pending_review') return 'border-amber-200 bg-amber-50 text-amber-700'
  if (status === 'rejected') return 'border-zinc-200 bg-zinc-50 text-zinc-700'
  if (status === 'paid') return 'border-emerald-200 bg-emerald-50 text-emerald-700'
  return 'border-border bg-muted/30 text-muted-foreground'
}

const rejectWithdraw = async (row: AdminSubsiteWithdraw) => {
  const confirmed = await confirmAction({
    description: t('admin.subsites.withdraws.actions.rejectConfirm', { id: row.id }),
    variant: 'destructive',
  })
  if (!confirmed) return

  const reason = window.prompt(t('admin.subsites.withdraws.actions.rejectReasonPrompt')) ?? ''
  operating.value = true
  try {
    await adminAPI.rejectSubsiteWithdraw(row.id, { reason: reason.trim() || undefined })
    notifySuccess(t('admin.subsites.withdraws.actions.rejectSuccess'))
    await fetchRows(pagination.value.page)
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.withdraws.actions.rejectFailed'))
  } finally {
    operating.value = false
  }
}

const payWithdraw = async (row: AdminSubsiteWithdraw) => {
  const confirmed = await confirmAction({ description: t('admin.subsites.withdraws.actions.payConfirm', { id: row.id }) })
  if (!confirmed) return

  operating.value = true
  try {
    await adminAPI.paySubsiteWithdraw(row.id)
    notifySuccess(t('admin.subsites.withdraws.actions.paySuccess'))
    await fetchRows(pagination.value.page)
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.withdraws.actions.payFailed'))
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
      <h1 class="text-2xl font-semibold">{{ t('admin.subsites.withdraws.title') }}</h1>
    </div>

    <div class="rounded-xl border border-border bg-card p-4 shadow-sm">
      <div class="flex flex-wrap items-center gap-3">
        <div class="w-full md:w-56">
          <Input v-model="filters.keyword" :placeholder="t('admin.subsites.withdraws.filters.keyword')" @update:modelValue="debouncedSearch" />
        </div>
        <div class="w-full md:w-44">
          <Select v-model="filters.status" @update:modelValue="handleSearch">
            <SelectTrigger class="h-9 w-full">
              <SelectValue :placeholder="t('admin.subsites.withdraws.filters.statusAll')" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">{{ t('admin.subsites.withdraws.filters.statusAll') }}</SelectItem>
              <SelectItem value="pending_review">{{ t('admin.subsites.withdraws.status.pending_review') }}</SelectItem>
              <SelectItem value="rejected">{{ t('admin.subsites.withdraws.status.rejected') }}</SelectItem>
              <SelectItem value="paid">{{ t('admin.subsites.withdraws.status.paid') }}</SelectItem>
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
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.id') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.site') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.amount') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.channel') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.account') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.status') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.rejectReason') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.subsites.withdraws.table.createdAt') }}</TableHead>
            <TableHead class="px-6 py-3 text-right">{{ t('admin.subsites.withdraws.table.action') }}</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody class="divide-y divide-border">
          <TableRow v-if="loading">
            <TableCell :colspan="9" class="p-0"><TableSkeleton :columns="9" :rows="5" /></TableCell>
          </TableRow>
          <TableRow v-else-if="rows.length === 0">
            <TableCell colspan="9" class="px-6 py-8 text-center text-muted-foreground">{{ t('admin.subsites.withdraws.empty') }}</TableCell>
          </TableRow>
          <TableRow v-for="item in rows" :key="item.id">
            <TableCell class="px-6 py-4"><IdCell :value="item.id" /></TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">
              <div class="text-foreground">{{ item.site?.site_name || '-' }}</div>
              <div class="font-mono">#{{ item.site_id }}</div>
            </TableCell>
            <TableCell class="px-6 py-4 font-mono text-xs">{{ item.amount || '0.00' }}</TableCell>
            <TableCell class="px-6 py-4 text-xs">{{ item.channel || '-' }}</TableCell>
            <TableCell class="px-6 py-4 text-xs break-all text-muted-foreground">{{ item.account || '-' }}</TableCell>
            <TableCell class="px-6 py-4 text-xs"><span class="inline-flex rounded-full border px-2.5 py-1" :class="statusClass(item.status)">{{ statusLabel(item.status) }}</span></TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ item.reject_reason || '-' }}</TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ formatDate(item.created_at) }}</TableCell>
            <TableCell class="px-6 py-4 text-right">
              <div class="flex justify-end gap-2">
                <Button size="sm" variant="outline" :disabled="operating || item.status !== 'pending_review'" @click="rejectWithdraw(item)">
                  {{ t('admin.subsites.withdraws.actions.reject') }}
                </Button>
                <Button size="sm" :disabled="operating || item.status !== 'pending_review'" @click="payWithdraw(item)">
                  {{ t('admin.subsites.withdraws.actions.pay') }}
                </Button>
              </div>
            </TableCell>
          </TableRow>
        </TableBody>
      </Table>
    </div>
  </div>
</template>
