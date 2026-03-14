<script setup lang="ts">
import { onMounted, reactive, ref, watch } from 'vue'
import { useDebounceFn } from '@vueuse/core'
import { useRoute } from 'vue-router'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import type { AdminOrder, AdminOrderItem } from '@/api/types'
import IdCell from '@/components/IdCell.vue'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import TableSkeleton from '@/components/TableSkeleton.vue'
import {
  orderStatusClass,
  orderStatusLabel,
} from '@/utils/status'
import { formatDate, formatMoney, toRFC3339 } from '@/utils/format'
import OrderDetailDialog from './components/OrderDetailDialog.vue'
import OrderFulfillmentModal from './components/OrderFulfillmentModal.vue'

const loading = ref(true)
const orders = ref<AdminOrder[]>([])
const pagination = ref({
  page: 1,
  page_size: 20,
  total: 0,
  total_page: 1,
})
const jumpPage = ref('')
const filters = reactive({
  orderNo: '',
  guestEmail: '',
  createdFrom: '',
  createdTo: '',
  status: '',
  userId: '',
  userKeyword: '',
})
const statusEdits = reactive<Record<number, string>>({})
const showDetail = ref(false)
const showFulfillmentModal = ref(false)
const selectedOrder = ref<AdminOrder | null>(null)
const fulfillmentParentId = ref<number | null>(null)
const route = useRoute()
const { t } = useI18n()
const adminPath = import.meta.env.VITE_ADMIN_PATH || ''
const userDetailLink = (userId: number) => `${adminPath}/users/${userId}`

const normalizeFilterValue = (value: string) => (value === '__all__' ? '' : value)

const toQueryText = (value: unknown) => {
  if (Array.isArray(value)) return String(value[0] || '').trim()
  if (value === undefined || value === null) return ''
  return String(value).trim()
}

const fetchOrders = async (page = 1) => {
  loading.value = true
  try {
    const response = await adminAPI.getOrders({
      page,
      page_size: pagination.value.page_size,
      status: normalizeFilterValue(filters.status) || undefined,
      user_id: filters.userId || undefined,
      user_keyword: filters.userKeyword || undefined,
      order_no: filters.orderNo || undefined,
      guest_email: filters.guestEmail || undefined,
      created_from: toRFC3339(filters.createdFrom),
      created_to: toRFC3339(filters.createdTo),
    })
    orders.value = response.data.data || []
    pagination.value = response.data.pagination || pagination.value
    orders.value.forEach((order) => {
      statusEdits[order.id] = order.status
    })
  } catch (error) {
    orders.value = []
  } finally {
    loading.value = false
  }
}

const handleSearch = () => {
  fetchOrders(1)
}
const debouncedSearch = useDebounceFn(handleSearch, 300)

const refresh = () => {
  fetchOrders(pagination.value.page)
}

const changePage = (page: number) => {
  if (page < 1 || page > pagination.value.total_page) return
  fetchOrders(page)
}

const jumpToPage = () => {
  if (!jumpPage.value) return
  const raw = Number(jumpPage.value)
  if (Number.isNaN(raw)) return
  const target = Math.min(Math.max(Math.floor(raw), 1), pagination.value.total_page)
  if (target === pagination.value.page) return
  changePage(target)
}

const canUpdateStatus = (order: AdminOrder) => {
  if (!order) return false
  return order.status !== 'completed' && order.status !== 'canceled'
}

const updateStatus = async (order: AdminOrder) => {
  if (!canUpdateStatus(order)) return
  const status = statusEdits[order.id]
  if (!status || status === order.status) return
  await adminAPI.updateOrderStatus(order.id, { status })
  fetchOrders(pagination.value.page)
}

const markCompleted = async (order: AdminOrder) => {
  if (!order || order.status !== 'delivered') return
  await adminAPI.updateOrderStatus(order.id, { status: 'completed' })
  fetchOrders(pagination.value.page)
}

const canCreateFulfillment = (order: AdminOrder | null) => {
  if (!order) return false
  if (order.fulfillment) return false
  if (order.parent_id == null && Array.isArray(order.children) && order.children.length > 0) return false
  if (Array.isArray(order.items) && order.items.length > 0 && !order.items.every((item: AdminOrderItem) => item.fulfillment_type === 'manual')) {
    return false
  }
  return order.status === 'paid' || order.status === 'fulfilling'
}

const openDetail = (order: AdminOrder) => {
  showFulfillmentModal.value = false
  selectedOrder.value = order
  showDetail.value = true
}

const openDetailById = (orderId: number) => {
  if (!orderId || orderId <= 0) return
  showFulfillmentModal.value = false
  selectedOrder.value = { id: orderId } as AdminOrder
  showDetail.value = true
}

const openFulfillment = (order: AdminOrder, parentId?: number) => {
  showDetail.value = false
  fulfillmentParentId.value = parentId || null
  selectedOrder.value = order
  showFulfillmentModal.value = true
}

const handleDetailClose = (value: boolean) => {
  showDetail.value = value
  if (!value) {
    selectedOrder.value = null
  }
}

const handleFulfillmentClose = (value: boolean) => {
  showFulfillmentModal.value = value
  if (!value) {
    selectedOrder.value = null
    fulfillmentParentId.value = null
  }
}

const handleFulfillmentSuccess = (parentId?: number | null) => {
  showFulfillmentModal.value = false
  fulfillmentParentId.value = null
  if (parentId) {
    // Re-open detail dialog for the parent order
    selectedOrder.value = { id: parentId } as AdminOrder
    showDetail.value = true
  }
  fetchOrders(pagination.value.page)
}

const handleDetailOpenFulfillment = (order: AdminOrder, parentId?: number) => {
  openFulfillment(order, parentId)
}

const statusLabel = (status: string) => orderStatusLabel(t, status)

const statusClass = (status: string) => orderStatusClass(status)

onMounted(() => {
  const initialUserId = toQueryText(route.query.user_id)
  filters.userId = initialUserId

  fetchOrders()

  const orderId = Number(route.query.order_id)
  if (Number.isFinite(orderId) && orderId > 0) {
    openDetailById(orderId)
  }
})

watch(
  () => route.query.order_id,
  (value) => {
    const orderId = Number(value)
    if (Number.isFinite(orderId) && orderId > 0) {
      openDetailById(orderId)
    }
  }
)

watch(
  () => route.query.user_id,
  (value) => {
    filters.userId = toQueryText(value)
    fetchOrders(1)
  }
)
</script>

<template>
  <div class="space-y-6">
    <div class="flex items-center justify-between">
      <div>
        <h1 class="text-2xl font-semibold">{{ t('admin.orders.title') }}</h1>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card p-4 shadow-sm">
      <div class="flex flex-wrap items-center gap-3">
        <div class="w-full md:w-32">
          <Input v-model="filters.userId" :placeholder="t('admin.orders.filterUserId')" @update:modelValue="debouncedSearch" />
        </div>
        <div class="w-full md:w-56">
          <Input v-model="filters.userKeyword" :placeholder="t('admin.orders.filterUserKeyword')" @update:modelValue="debouncedSearch" />
        </div>
        <div class="w-full md:w-48">
          <Input v-model="filters.orderNo" :placeholder="t('admin.orders.filterOrderNo')" @update:modelValue="debouncedSearch" />
        </div>
        <div class="w-full md:w-48">
          <Input v-model="filters.guestEmail" :placeholder="t('admin.orders.filterGuestEmail')" @update:modelValue="debouncedSearch" />
        </div>
        <div class="flex flex-wrap items-center gap-2">
          <span class="text-xs text-muted-foreground whitespace-nowrap">{{ t('admin.orders.filterCreatedRange') }}</span>
          <Input
            v-model="filters.createdFrom"
            type="datetime-local"
            class="h-9 w-full md:w-auto"
            :placeholder="t('admin.orders.filterCreatedFrom')"
            @update:modelValue="handleSearch"
          />
          <span class="text-muted-foreground">-</span>
          <Input
            v-model="filters.createdTo"
            type="datetime-local"
            class="h-9 w-full md:w-auto"
            :placeholder="t('admin.orders.filterCreatedTo')"
            @update:modelValue="handleSearch"
          />
        </div>
        <div class="w-full md:w-40">
          <Select v-model="filters.status" @update:modelValue="handleSearch">
            <SelectTrigger class="h-9 w-full">
              <SelectValue :placeholder="t('admin.orders.filterStatusAll')" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="__all__">{{ t('admin.orders.filterStatusAll') }}</SelectItem>
              <SelectItem value="pending_payment">{{ t('order.status.pending_payment') }}</SelectItem>
              <SelectItem value="paid">{{ t('order.status.paid') }}</SelectItem>
              <SelectItem value="fulfilling">{{ t('order.status.fulfilling') }}</SelectItem>
              <SelectItem value="partially_delivered">{{ t('order.status.partially_delivered') }}</SelectItem>
              <SelectItem value="delivered">{{ t('order.status.delivered') }}</SelectItem>
              <SelectItem value="completed">{{ t('order.status.completed') }}</SelectItem>
              <SelectItem value="canceled">{{ t('order.status.canceled') }}</SelectItem>
            </SelectContent>
          </Select>
        </div>
        <div class="flex-1"></div>
        <Button size="sm" @click="refresh">{{ t('admin.common.refresh') }}</Button>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card">
      <Table>
        <TableHeader class="bg-muted/40 text-xs uppercase text-muted-foreground">
          <TableRow>
            <TableHead class="px-6 py-3">{{ t('admin.orders.table.id') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.orders.table.orderNo') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.orders.table.user') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.orders.table.ip') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.orders.table.amount') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.orders.table.status') }}</TableHead>
            <TableHead class="px-6 py-3">{{ t('admin.orders.table.createdAt') }}</TableHead>
            <TableHead class="px-6 py-3 text-right">{{ t('admin.orders.table.action') }}</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody class="divide-y divide-border">
          <TableRow v-if="loading">
            <TableCell :colspan="8" class="p-0">
              <TableSkeleton :columns="8" :rows="5" />
            </TableCell>
          </TableRow>
          <TableRow v-else-if="orders.length === 0">
            <TableCell colspan="8" class="px-6 py-8 text-center text-muted-foreground">{{ t('admin.orders.empty') }}</TableCell>
          </TableRow>
          <TableRow v-for="order in orders" :key="order.id" class="hover:bg-muted/30">
            <TableCell class="px-6 py-4">
              <IdCell :value="order.id" />
            </TableCell>
            <TableCell class="px-6 py-4">
              <div class="font-medium text-foreground">{{ order.order_no }}</div>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">
              <div v-if="order.user_id">
                <div class="text-foreground">{{ order.user_display_name || '-' }}</div>
                <div class="text-muted-foreground">{{ order.user_email || '-' }}</div>
                <div class="mt-1">
                  {{ t('admin.orders.userLabel') }}:
                  <a :href="userDetailLink(order.user_id)" target="_blank" rel="noopener" class="text-primary underline-offset-4 hover:underline">
                    #{{ order.user_id }}
                  </a>
                </div>
              </div>
              <div v-else>{{ t('admin.orders.guestLabel') }}: {{ order.guest_email || '-' }}</div>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ order.client_ip || '-' }}</TableCell>
            <TableCell class="px-6 py-4 font-mono text-foreground">{{ formatMoney(order.total_amount, order.currency) }}</TableCell>
            <TableCell class="px-6 py-4">
              <span class="inline-flex rounded-full border px-2.5 py-1 text-xs" :class="statusClass(order.status)">
                {{ statusLabel(order.status) }}
              </span>
            </TableCell>
            <TableCell class="px-6 py-4 text-xs text-muted-foreground">{{ formatDate(order.created_at) }}</TableCell>
            <TableCell class="px-6 py-4">
              <div class="flex flex-wrap items-center justify-end gap-2">
                <Select v-if="canUpdateStatus(order)" v-model="statusEdits[order.id]">
                  <SelectTrigger class="h-8 w-[150px] text-xs">
                    <SelectValue :placeholder="t('admin.orders.filterStatusAll')" />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="pending_payment">{{ t('order.status.pending_payment') }}</SelectItem>
                    <SelectItem value="paid">{{ t('order.status.paid') }}</SelectItem>
                    <SelectItem value="fulfilling">{{ t('order.status.fulfilling') }}</SelectItem>
                    <SelectItem value="delivered">{{ t('order.status.delivered') }}</SelectItem>
                    <SelectItem value="completed">{{ t('order.status.completed') }}</SelectItem>
                    <SelectItem value="canceled">{{ t('order.status.canceled') }}</SelectItem>
                  </SelectContent>
                </Select>
                <Button v-if="canUpdateStatus(order)" size="sm" variant="outline" @click="updateStatus(order)">
                  {{ t('admin.orders.update') }}
                </Button>
                <Button v-if="canCreateFulfillment(order)" size="sm" variant="secondary" @click="openFulfillment(order)">
                  {{ t('admin.orders.fulfillmentCreate') }}
                </Button>
                <Button v-if="order.status === 'delivered'" size="sm" variant="outline" @click="markCompleted(order)">
                  {{ t('admin.orders.markCompleted') }}
                </Button>
                <Button size="sm" variant="outline" @click="openDetail(order)">
                  {{ t('admin.orders.view') }}
                </Button>
              </div>
            </TableCell>
          </TableRow>
        </TableBody>
      </Table>

      <div
        v-if="pagination.total_page > 1"
        class="flex flex-wrap items-center justify-between gap-3 border-t border-border px-6 py-4"
      >
        <div class="flex items-center gap-3">
          <span class="text-xs text-muted-foreground">
            {{ t('admin.common.pageInfo', { total: pagination.total, page: pagination.page, totalPage: pagination.total_page }) }}
          </span>
        </div>
        <div class="flex flex-wrap items-center gap-2">
          <div class="flex items-center gap-2">
            <Input
              v-model="jumpPage"
              type="number"
              min="1"
              :max="pagination.total_page"
              class="h-8 w-20"
              :placeholder="t('admin.common.jumpPlaceholder')"
            />
            <Button variant="outline" size="sm" class="h-8" @click="jumpToPage">
              {{ t('admin.common.jumpTo') }}
            </Button>
          </div>
          <div class="flex items-center gap-2">
            <Button variant="outline" size="sm" class="h-8" :disabled="pagination.page <= 1" @click="changePage(pagination.page - 1)">
              {{ t('admin.common.prevPage') }}
            </Button>
            <Button
              variant="outline"
              size="sm"
              class="h-8"
              :disabled="pagination.page >= pagination.total_page"
              @click="changePage(pagination.page + 1)"
            >
              {{ t('admin.common.nextPage') }}
            </Button>
          </div>
        </div>
      </div>
    </div>

    <OrderDetailDialog
      :model-value="showDetail"
      :order="selectedOrder"
      site-currency=""
      @update:model-value="handleDetailClose"
      @refresh="refresh"
      @open-fulfillment="handleDetailOpenFulfillment"
    />

    <OrderFulfillmentModal
      :model-value="showFulfillmentModal"
      :order="selectedOrder"
      site-currency=""
      :parent-id="fulfillmentParentId"
      @update:model-value="handleFulfillmentClose"
      @success="handleFulfillmentSuccess"
    />
  </div>
</template>
