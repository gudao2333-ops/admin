<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import { adminAPI } from '@/api/admin'
import type { AdminProduct, AdminRedeemCodeBatch } from '@/api/types'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import TableSkeleton from '@/components/TableSkeleton.vue'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { formatDate, getLocalizedText } from '@/utils/format'

const loading = ref(true)
const products = ref<AdminProduct[]>([])
const batches = ref<AdminRedeemCodeBatch[]>([])
const formError = ref('')
const createSubmitting = ref(false)
const generateSubmittingID = ref<number | null>(null)

const filters = reactive({
  name: '',
  product_id: '__all__',
  sku_id: '__all__',
})

const createForm = reactive({
  name: '',
  product_id: '',
  sku_id: '',
  quantity: 1,
  expires_at: '',
  remark: '',
  generate_count: 10,
})

const skuOptions = ref<Array<{ id: number; label: string }>>([])
const filterSkuOptions = ref<Array<{ id: number; label: string }>>([])

const parseID = (raw: string) => {
  const n = Number(raw)
  if (!Number.isFinite(n) || n <= 0) return 0
  return Math.floor(n)
}

const refreshSkuOptions = () => {
  const productID = parseID(createForm.product_id)
  const selected = products.value.find((item) => Number(item.id) === productID)
  const skus = Array.isArray(selected?.skus) ? selected!.skus : []
  skuOptions.value = skus
    .filter((sku) => Number(sku?.id || 0) > 0)
    .map((sku) => {
      const spec = Object.entries(sku.spec_values || {})
        .map(([k, v]) => `${k}:${v}`)
        .join(' / ')
      const label = [sku.sku_code, spec].filter(Boolean).join(' | ')
      return { id: Number(sku.id), label: label || `#${sku.id}` }
    })
  if (!skuOptions.value.some((item) => item.id === parseID(createForm.sku_id))) {
    createForm.sku_id = ''
  }
}

const refreshFilterSkuOptions = () => {
  const productID = parseID(filters.product_id)
  const selected = products.value.find((item) => Number(item.id) === productID)
  const skus = Array.isArray(selected?.skus) ? selected!.skus : []
  filterSkuOptions.value = skus
    .filter((sku) => Number(sku?.id || 0) > 0)
    .map((sku) => {
      const spec = Object.entries(sku.spec_values || {})
        .map(([k, v]) => `${k}:${v}`)
        .join(' / ')
      const label = [sku.sku_code, spec].filter(Boolean).join(' | ')
      return { id: Number(sku.id), label: label || `#${sku.id}` }
    })
  if (!filterSkuOptions.value.some((item) => String(item.id) === filters.sku_id)) {
    filters.sku_id = '__all__'
  }
}

const handleFilterProductChange = async (value: unknown) => {
  filters.product_id = typeof value === 'string' ? value : '__all__'
  refreshFilterSkuOptions()
  await loadBatches()
}

const loadProducts = async () => {
  const response = await adminAPI.getProducts({ page: 1, page_size: 200 })
  products.value = (response.data.data || []) as AdminProduct[]
}

const loadBatches = async () => {
  loading.value = true
  try {
    const params: Record<string, unknown> = {
      page: 1,
      page_size: 200,
      name: filters.name.trim() || undefined,
      product_id: filters.product_id === '__all__' ? undefined : parseID(filters.product_id),
      sku_id: filters.sku_id === '__all__' ? undefined : parseID(filters.sku_id),
    }
    const response = await adminAPI.getRedeemCodeBatches(params)
    batches.value = (response.data.data || []) as AdminRedeemCodeBatch[]
  } finally {
    loading.value = false
  }
}

const createBatch = async () => {
  formError.value = ''
  const productID = parseID(createForm.product_id)
  const skuID = parseID(createForm.sku_id)
  if (!createForm.name.trim() || !productID || !skuID || Number(createForm.quantity) <= 0) {
    formError.value = '请完整填写批次信息'
    return
  }
  createSubmitting.value = true
  try {
    const payload = {
      name: createForm.name.trim(),
      product_id: productID,
      sku_id: skuID,
      quantity: Math.floor(Number(createForm.quantity)),
      expires_at: createForm.expires_at ? new Date(createForm.expires_at).toISOString() : '',
      remark: createForm.remark.trim(),
    }
    const response = await adminAPI.createRedeemCodeBatch(payload)
    const batchID = Number(response.data.data?.id || 0)
    const generateCount = Math.floor(Number(createForm.generate_count) || 0)
    if (batchID > 0 && generateCount > 0) {
      await adminAPI.generateRedeemCodes(batchID, { count: generateCount })
    }
    createForm.name = ''
    createForm.remark = ''
    createForm.expires_at = ''
    createForm.quantity = 1
    createForm.generate_count = 10
    await loadBatches()
  } catch (err: any) {
    formError.value = err?.message || '创建批次失败'
  } finally {
    createSubmitting.value = false
  }
}

const generateCodes = async (batch: AdminRedeemCodeBatch) => {
  const count = Math.floor(Number(createForm.generate_count) || 0)
  if (count <= 0) return
  generateSubmittingID.value = Number(batch.id)
  try {
    await adminAPI.generateRedeemCodes(Number(batch.id), { count })
    await loadBatches()
  } finally {
    generateSubmittingID.value = null
  }
}

onMounted(async () => {
  await loadProducts()
  refreshSkuOptions()
  refreshFilterSkuOptions()
  await loadBatches()
})
</script>

<template>
  <div class="space-y-6">
    <h1 class="text-2xl font-semibold">兑换码批次</h1>

    <div class="rounded-xl border border-border bg-card p-4 space-y-3">
      <div class="grid grid-cols-1 gap-3 md:grid-cols-12">
        <div class="md:col-span-4">
          <Input v-model="createForm.name" placeholder="批次名称（可用渠道名）" />
        </div>
        <div class="md:col-span-2">
          <Select v-model="createForm.product_id" @update:modelValue="refreshSkuOptions">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="选择商品" /></SelectTrigger>
            <SelectContent>
              <SelectItem v-for="product in products" :key="product.id" :value="String(product.id)">
                #{{ product.id }} {{ getLocalizedText(product.title || {}) }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-2">
          <Select v-model="createForm.sku_id">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="选择 SKU" /></SelectTrigger>
            <SelectContent>
              <SelectItem v-for="sku in skuOptions" :key="sku.id" :value="String(sku.id)">{{ sku.label }}</SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-1"><Input v-model.number="createForm.quantity" type="number" min="1" placeholder="数量" /></div>
        <div class="md:col-span-1"><Input v-model.number="createForm.generate_count" type="number" min="1" placeholder="生码数" /></div>
        <div class="md:col-span-2"><Input v-model="createForm.expires_at" type="datetime-local" /></div>
      </div>
      <Input v-model="createForm.remark" placeholder="备注" />
      <div class="flex items-center gap-3">
        <Button size="sm" :disabled="createSubmitting" @click="createBatch">{{ createSubmitting ? '提交中...' : '创建批次并生码' }}</Button>
        <span v-if="formError" class="text-sm text-destructive">{{ formError }}</span>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card p-4 space-y-3">
      <div class="grid grid-cols-1 gap-3 md:grid-cols-12">
        <div class="md:col-span-4"><Input v-model="filters.name" placeholder="批次名筛选" @keyup.enter="loadBatches" /></div>
        <div class="md:col-span-3">
          <Select v-model="filters.product_id" @update:modelValue="handleFilterProductChange">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="商品筛选" /></SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">全部商品</SelectItem>
              <SelectItem v-for="product in products" :key="product.id" :value="String(product.id)">
                #{{ product.id }} {{ getLocalizedText(product.title || {}) }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-3">
          <Select v-model="filters.sku_id" @update:modelValue="loadBatches">
            <SelectTrigger class="h-9 w-full"><SelectValue placeholder="SKU筛选" /></SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">全部 SKU</SelectItem>
              <SelectItem v-for="sku in filterSkuOptions" :key="sku.id" :value="String(sku.id)">
                {{ sku.label }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="md:col-span-2"><Button size="sm" variant="outline" @click="loadBatches">搜索</Button></div>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card">
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>ID</TableHead>
            <TableHead>批次名称</TableHead>
            <TableHead>商品</TableHead>
            <TableHead>SKU</TableHead>
            <TableHead>单码数量</TableHead>
            <TableHead>已生成</TableHead>
            <TableHead>有效期</TableHead>
            <TableHead>备注</TableHead>
            <TableHead>创建时间</TableHead>
            <TableHead class="text-right">操作</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          <TableRow v-if="loading">
            <TableCell :colspan="10" class="p-0"><TableSkeleton :columns="10" :rows="4" /></TableCell>
          </TableRow>
          <TableRow v-else-if="batches.length === 0"><TableCell :colspan="10" class="text-center py-8">暂无数据</TableCell></TableRow>
          <TableRow v-for="batch in batches" :key="batch.id">
            <TableCell>{{ batch.id }}</TableCell>
            <TableCell>{{ batch.name }}</TableCell>
            <TableCell>#{{ batch.product_id }} {{ getLocalizedText(batch.product?.title || {}) }}</TableCell>
            <TableCell>#{{ batch.sku_id }} {{ batch.sku?.sku_code || '-' }}</TableCell>
            <TableCell>{{ batch.quantity }}</TableCell>
            <TableCell>{{ batch.generated_count }}</TableCell>
            <TableCell>{{ formatDate(batch.expires_at) || '-' }}</TableCell>
            <TableCell>{{ batch.remark || '-' }}</TableCell>
            <TableCell>{{ formatDate(batch.created_at) || '-' }}</TableCell>
            <TableCell class="text-right">
              <Button size="sm" variant="outline" :disabled="generateSubmittingID === batch.id" @click="generateCodes(batch)">
                {{ generateSubmittingID === batch.id ? '生成中...' : '追加生码' }}
              </Button>
            </TableCell>
          </TableRow>
        </TableBody>
      </Table>
    </div>
  </div>
</template>
