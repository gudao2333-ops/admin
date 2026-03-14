<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'
import { adminAPI } from '@/api/admin'
import type { AdminProduct, AdminRedeemCode, AdminRedeemCodeBatch } from '@/api/types'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import TableSkeleton from '@/components/TableSkeleton.vue'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { formatDate, getLocalizedText } from '@/utils/format'

const loading = ref(true)
const products = ref<AdminProduct[]>([])
const batches = ref<AdminRedeemCodeBatch[]>([])
const rows = ref<AdminRedeemCode[]>([])
const actionLoadingID = ref<number | null>(null)
const errorText = ref('')

const filters = reactive({
  code: '',
  status: '__all__',
  product_id: '__all__',
  sku_id: '__all__',
  batch_id: '__all__',
})

const parseID = (raw: string) => {
  const n = Number(raw)
  if (!Number.isFinite(n) || n <= 0) return 0
  return Math.floor(n)
}

const formatSkuLabel = (sku: any) => {
  const spec = Object.entries(sku?.spec_values || {})
    .map(([k, v]) => `${k}:${v}`)
    .join(' / ')
  return [sku?.sku_code, spec].filter(Boolean).join(' | ') || `#${sku?.id || '-'}`
}

const skuOptions = computed(() => {
  const productID = parseID(filters.product_id)
  if (!productID) return [] as Array<{ id: number; label: string }>
  const selected = products.value.find((item) => Number(item.id) === productID)
  const skus = Array.isArray(selected?.skus) ? selected!.skus : []
  return skus
    .filter((sku) => Number(sku?.id || 0) > 0)
    .map((sku) => ({ id: Number(sku.id), label: formatSkuLabel(sku) }))
})

const handleProductChange = async (value: unknown) => {
  const selectedValue = typeof value === 'string' ? value : '__all__'
  filters.product_id = selectedValue
  if (!skuOptions.value.some((sku) => String(sku.id) === filters.sku_id)) {
    filters.sku_id = '__all__'
  }
  await loadCodes()
}

const loadProducts = async () => {
  const response = await adminAPI.getProducts({ page: 1, page_size: 200 })
  products.value = (response.data.data || []) as AdminProduct[]
}

const loadBatches = async () => {
  const response = await adminAPI.getRedeemCodeBatches({ page: 1, page_size: 500 })
  batches.value = (response.data.data || []) as AdminRedeemCodeBatch[]
}

const loadCodes = async () => {
  loading.value = true
  errorText.value = ''
  try {
    const params: Record<string, unknown> = {
      page: 1,
      page_size: 500,
      code: filters.code.trim() || undefined,
      status: filters.status === '__all__' ? undefined : filters.status,
      product_id: filters.product_id === '__all__' ? undefined : parseID(filters.product_id),
      sku_id: filters.sku_id === '__all__' ? undefined : parseID(filters.sku_id),
      batch_id: filters.batch_id === '__all__' ? undefined : parseID(filters.batch_id),
    }
    const response = await adminAPI.getRedeemCodes(params)
    rows.value = (response.data.data || []) as AdminRedeemCode[]
  } catch (err: any) {
    rows.value = []
    errorText.value = err?.message || '加载失败'
  } finally {
    loading.value = false
  }
}

const toggleStatus = async (row: AdminRedeemCode) => {
  const nextStatus = row.status === 'frozen' ? 'unused' : 'frozen'
  actionLoadingID.value = Number(row.id)
  errorText.value = ''
  try {
    await adminAPI.updateRedeemCodeStatus(Number(row.id), { status: nextStatus })
    await loadCodes()
  } catch (err: any) {
    errorText.value = err?.message || '更新状态失败'
  } finally {
    actionLoadingID.value = null
  }
}

const exportCodes = async (format: 'csv' | 'txt') => {
  const payload = {
    code: filters.code.trim() || undefined,
    status: filters.status === '__all__' ? undefined : filters.status,
    product_id: filters.product_id === '__all__' ? undefined : parseID(filters.product_id),
    sku_id: filters.sku_id === '__all__' ? undefined : parseID(filters.sku_id),
    batch_id: filters.batch_id === '__all__' ? undefined : parseID(filters.batch_id),
    format,
  }
  const response = await adminAPI.exportRedeemCodes(payload) as { data: string; headers?: Record<string, string> }
  const contentDisposition = String(response?.headers?.['content-disposition'] || '')
  const filenameMatch = contentDisposition.match(/filename="?([^";]+)"?/i)
  const fallbackName = `redeem-codes-${new Date().toISOString().replace(/[:.]/g, '-')}.${format}`
  const filename = filenameMatch?.[1] || fallbackName
  const contentType = format === 'csv' ? 'text/csv;charset=utf-8' : 'text/plain;charset=utf-8'
  const blob = new Blob([response.data], { type: contentType })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = filename
  document.body.appendChild(link)
  link.click()
  link.remove()
  URL.revokeObjectURL(url)
}

const statusText = (row: AdminRedeemCode) => {
  if (row.is_expired && row.status === 'unused') return 'expired'
  return row.status
}

onMounted(async () => {
  await Promise.all([loadProducts(), loadBatches()])
  await loadCodes()
})
</script>

<template>
  <div class="space-y-6">
    <h1 class="text-2xl font-semibold">兑换码列表</h1>

    <div class="rounded-xl border border-border bg-card p-4 space-y-3">
      <div class="grid grid-cols-1 gap-3 md:grid-cols-12">
        <div class="md:col-span-3"><Input v-model="filters.code" placeholder="兑换码" @keyup.enter="loadCodes" /></div>
        <div class="md:col-span-2">
          <Select v-model="filters.status" @update:modelValue="loadCodes">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="状态" /></SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">全部状态</SelectItem>
              <SelectItem value="unused">unused</SelectItem>
              <SelectItem value="used">used</SelectItem>
              <SelectItem value="frozen">frozen</SelectItem>
              <SelectItem value="expired">expired</SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-2">
          <Select v-model="filters.product_id" @update:modelValue="handleProductChange">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="商品" /></SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">全部商品</SelectItem>
              <SelectItem v-for="product in products" :key="product.id" :value="String(product.id)">
                #{{ product.id }} {{ getLocalizedText(product.title || {}) }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-2">
          <Select v-model="filters.sku_id" @update:modelValue="loadCodes">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="SKU" /></SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">全部 SKU</SelectItem>
              <SelectItem v-for="sku in skuOptions" :key="sku.id" :value="String(sku.id)">
                {{ sku.label }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-2">
          <Select v-model="filters.batch_id" @update:modelValue="loadCodes">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="批次" /></SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">全部批次</SelectItem>
              <SelectItem v-for="batch in batches" :key="batch.id" :value="String(batch.id)">
                #{{ batch.id }} {{ batch.name }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-2 flex gap-2">
          <Button size="sm" variant="outline" @click="loadCodes">搜索</Button>
          <Button size="sm" variant="outline" @click="exportCodes('csv')">导出 CSV</Button>
        </div>
      </div>
      <div v-if="errorText" class="text-sm text-destructive">{{ errorText }}</div>
    </div>

    <div class="rounded-xl border border-border bg-card">
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>ID</TableHead>
            <TableHead>Code</TableHead>
            <TableHead>批次</TableHead>
            <TableHead>商品</TableHead>
            <TableHead>SKU</TableHead>
            <TableHead>状态</TableHead>
            <TableHead>兑换用户</TableHead>
            <TableHead>兑换订单</TableHead>
            <TableHead>兑换时间</TableHead>
            <TableHead>创建时间</TableHead>
            <TableHead class="text-right">操作</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          <TableRow v-if="loading"><TableCell :colspan="11" class="p-0"><TableSkeleton :columns="11" :rows="5" /></TableCell></TableRow>
          <TableRow v-else-if="rows.length === 0"><TableCell :colspan="11" class="text-center py-8">暂无数据</TableCell></TableRow>
          <TableRow v-for="row in rows" :key="row.id">
            <TableCell>{{ row.id }}</TableCell>
            <TableCell class="font-mono text-xs">{{ row.code }}</TableCell>
            <TableCell>#{{ row.batch_id }} {{ row.batch?.name || '-' }}</TableCell>
            <TableCell>#{{ row.batch?.product_id || '-' }} {{ getLocalizedText(row.batch?.product?.title || {}) }}</TableCell>
            <TableCell>#{{ row.batch?.sku_id || '-' }} {{ row.batch?.sku?.sku_code || '-' }}</TableCell>
            <TableCell>{{ statusText(row) }}</TableCell>
            <TableCell>
              <span v-if="row.used_user">#{{ row.used_user.id }} {{ row.used_user.display_name || row.used_user.email || '-' }}</span>
              <span v-else>-</span>
            </TableCell>
            <TableCell>
              <span v-if="row.used_order">#{{ row.used_order.id }} {{ row.used_order.order_no || '-' }}</span>
              <span v-else>-</span>
            </TableCell>
            <TableCell>{{ formatDate(row.used_at) || '-' }}</TableCell>
            <TableCell>{{ formatDate(row.created_at) || '-' }}</TableCell>
            <TableCell class="text-right">
              <Button
                v-if="row.status !== 'used'"
                size="sm"
                variant="outline"
                :disabled="actionLoadingID === row.id"
                @click="toggleStatus(row)"
              >
                {{ row.status === 'frozen' ? '恢复' : '冻结' }}
              </Button>
            </TableCell>
          </TableRow>
        </TableBody>
      </Table>
    </div>
  </div>
</template>
