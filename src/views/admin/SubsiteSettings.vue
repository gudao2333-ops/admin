<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import type { AdminSubsiteSettings, AdminSubsiteSuffix } from '@/api/types'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import { notifyError, notifySuccess } from '@/utils/notify'
import { confirmAction } from '@/utils/confirm'

const { t } = useI18n()
const loading = ref(false)
const suffixLoading = ref(false)
const savingSuffix = ref(false)

const form = reactive({
  enabled: false,
  opening_price: 0,
  profit_confirm_days: 0,
  min_withdraw_amount: 0,
  withdraw_channels_text: '',
  reserved_prefixes_text: '',
})

const suffixForm = reactive({
  suffix: '',
  sort_order: 0,
})

const suffixes = ref<AdminSubsiteSuffix[]>([])

const normalizeNumber = (value: unknown, fallback: number) => {
  const parsed = Number(value)
  if (Number.isNaN(parsed)) return fallback
  return parsed
}

const splitList = (value: string) =>
  value
    .split(/\r?\n|,/) 
    .map((item) => item.trim())
    .filter((item) => item !== '')

const joinList = (value: unknown) => {
  if (!Array.isArray(value)) return ''
  return value
    .map((item) => String(item || '').trim())
    .filter((item) => item !== '')
    .join('\n')
}

const hydrateSettings = (settings?: AdminSubsiteSettings) => {
  if (!settings) return
  form.enabled = Boolean(settings.enabled)
  form.opening_price = Math.max(normalizeNumber(settings.opening_price, 0), 0)
  form.profit_confirm_days = Math.max(normalizeNumber(settings.profit_confirm_days, 0), 0)
  form.min_withdraw_amount = Math.max(normalizeNumber(settings.min_withdraw_amount, 0), 0)
  form.withdraw_channels_text = joinList(settings.withdraw_channels)
  form.reserved_prefixes_text = joinList(settings.reserved_prefixes)
}

const fetchSettings = async () => {
  loading.value = true
  try {
    const response = await adminAPI.getSubsiteSettings()
    hydrateSettings(response.data?.data)
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.settings.loadFailed'))
  } finally {
    loading.value = false
  }
}

const saveSettings = async () => {
  loading.value = true
  const payload: AdminSubsiteSettings = {
    enabled: form.enabled,
    opening_price: Math.max(normalizeNumber(form.opening_price, 0), 0),
    profit_confirm_days: Math.max(Math.floor(normalizeNumber(form.profit_confirm_days, 0)), 0),
    min_withdraw_amount: Math.max(normalizeNumber(form.min_withdraw_amount, 0), 0),
    withdraw_channels: splitList(form.withdraw_channels_text),
    reserved_prefixes: splitList(form.reserved_prefixes_text),
  }
  try {
    const response = await adminAPI.updateSubsiteSettings(payload)
    hydrateSettings(response.data?.data || payload)
    notifySuccess(t('admin.subsites.settings.saveSuccess'))
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.settings.saveFailed'))
  } finally {
    loading.value = false
  }
}

const fetchSuffixes = async () => {
  suffixLoading.value = true
  try {
    const response = await adminAPI.getSubsiteSuffixes()
    suffixes.value = response.data?.data || []
  } catch {
    suffixes.value = []
  } finally {
    suffixLoading.value = false
  }
}

const addSuffix = async () => {
  const suffix = suffixForm.suffix.trim()
  if (!suffix) return
  savingSuffix.value = true
  try {
    await adminAPI.createSubsiteSuffix({
      suffix,
      sort_order: Math.floor(normalizeNumber(suffixForm.sort_order, 0)),
      is_active: true,
    })
    suffixForm.suffix = ''
    suffixForm.sort_order = 0
    notifySuccess(t('admin.subsites.suffixes.createSuccess'))
    await fetchSuffixes()
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.suffixes.createFailed'))
  } finally {
    savingSuffix.value = false
  }
}

const toggleSuffix = async (item: AdminSubsiteSuffix) => {
  try {
    await adminAPI.updateSubsiteSuffix(item.id, { is_active: !item.is_active })
    notifySuccess(t('admin.subsites.suffixes.updateSuccess'))
    await fetchSuffixes()
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.suffixes.updateFailed'))
  }
}

const removeSuffix = async (item: AdminSubsiteSuffix) => {
  const confirmed = await confirmAction({ description: t('admin.subsites.suffixes.deleteConfirm', { suffix: item.suffix }), variant: 'destructive' })
  if (!confirmed) return

  try {
    await adminAPI.deleteSubsiteSuffix(item.id)
    notifySuccess(t('admin.subsites.suffixes.deleteSuccess'))
    await fetchSuffixes()
  } catch (err: any) {
    notifyError(err?.message || t('admin.subsites.suffixes.deleteFailed'))
  }
}

onMounted(async () => {
  await Promise.all([fetchSettings(), fetchSuffixes()])
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-semibold">{{ t('admin.subsites.settings.title') }}</h1>
    </div>

    <div class="rounded-xl border border-border bg-card p-6">
      <p class="mb-6 text-sm text-muted-foreground">{{ t('admin.subsites.settings.subtitle') }}</p>
      <div class="space-y-6">
        <div class="flex items-center gap-3 rounded-lg border border-border bg-muted/20 px-4 py-3">
          <input id="subsite-enabled" v-model="form.enabled" type="checkbox" class="h-4 w-4 accent-primary" />
          <label for="subsite-enabled" class="text-sm font-medium">{{ t('admin.subsites.settings.enabled') }}</label>
        </div>

        <div class="grid grid-cols-1 gap-6 md:grid-cols-3">
          <div class="space-y-2">
            <label class="text-xs font-medium text-muted-foreground">{{ t('admin.subsites.settings.openingPrice') }}</label>
            <Input v-model.number="form.opening_price" type="number" min="0" step="0.01" />
          </div>
          <div class="space-y-2">
            <label class="text-xs font-medium text-muted-foreground">{{ t('admin.subsites.settings.profitConfirmDays') }}</label>
            <Input v-model.number="form.profit_confirm_days" type="number" min="0" step="1" />
          </div>
          <div class="space-y-2">
            <label class="text-xs font-medium text-muted-foreground">{{ t('admin.subsites.settings.minWithdrawAmount') }}</label>
            <Input v-model.number="form.min_withdraw_amount" type="number" min="0" step="0.01" />
          </div>
        </div>

        <div class="grid grid-cols-1 gap-6 md:grid-cols-2">
          <div class="space-y-2">
            <label class="text-xs font-medium text-muted-foreground">{{ t('admin.subsites.settings.withdrawChannels') }}</label>
            <Textarea v-model="form.withdraw_channels_text" rows="5" :placeholder="t('admin.subsites.settings.multilinePlaceholder')" />
          </div>
          <div class="space-y-2">
            <label class="text-xs font-medium text-muted-foreground">{{ t('admin.subsites.settings.reservedPrefixes') }}</label>
            <Textarea v-model="form.reserved_prefixes_text" rows="5" :placeholder="t('admin.subsites.settings.multilinePlaceholder')" />
          </div>
        </div>

        <div class="flex justify-end">
          <Button :disabled="loading" @click="saveSettings">{{ t('admin.common.save') }}</Button>
        </div>
      </div>
    </div>

    <div class="rounded-xl border border-border bg-card p-6">
      <h2 class="text-lg font-semibold">{{ t('admin.subsites.suffixes.title') }}</h2>
      <p class="mt-1 text-sm text-muted-foreground">{{ t('admin.subsites.suffixes.subtitle') }}</p>

      <div class="mt-4 flex flex-wrap items-end gap-3 rounded-lg border border-border bg-muted/20 p-4">
        <div class="w-full md:w-56">
          <label class="mb-1 block text-xs text-muted-foreground">{{ t('admin.subsites.suffixes.suffix') }}</label>
          <Input v-model="suffixForm.suffix" :placeholder="t('admin.subsites.suffixes.suffixPlaceholder')" />
        </div>
        <div class="w-full md:w-36">
          <label class="mb-1 block text-xs text-muted-foreground">{{ t('admin.subsites.suffixes.sortOrder') }}</label>
          <Input v-model.number="suffixForm.sort_order" type="number" min="0" step="1" />
        </div>
        <Button :disabled="savingSuffix" @click="addSuffix">{{ t('admin.common.create') }}</Button>
      </div>

      <div class="mt-4 rounded-lg border border-border">
        <Table>
          <TableHeader class="border-b border-border bg-muted/40 text-xs uppercase text-muted-foreground">
            <TableRow>
              <TableHead class="px-6 py-3">{{ t('admin.subsites.suffixes.suffix') }}</TableHead>
              <TableHead class="px-6 py-3">{{ t('admin.subsites.suffixes.sortOrder') }}</TableHead>
              <TableHead class="px-6 py-3">{{ t('admin.subsites.suffixes.status') }}</TableHead>
              <TableHead class="px-6 py-3 text-right">{{ t('admin.subsites.suffixes.actions') }}</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody class="divide-y divide-border">
            <TableRow v-if="suffixLoading">
              <TableCell colspan="4" class="px-6 py-6 text-center text-muted-foreground">{{ t('admin.common.loading') }}</TableCell>
            </TableRow>
            <TableRow v-else-if="suffixes.length === 0">
              <TableCell colspan="4" class="px-6 py-6 text-center text-muted-foreground">{{ t('admin.subsites.suffixes.empty') }}</TableCell>
            </TableRow>
            <TableRow v-for="item in suffixes" :key="item.id">
              <TableCell class="px-6 py-4 font-mono text-xs">{{ item.suffix }}</TableCell>
              <TableCell class="px-6 py-4 text-xs">{{ item.sort_order }}</TableCell>
              <TableCell class="px-6 py-4 text-xs">
                <span class="inline-flex rounded-full border px-2.5 py-1" :class="item.is_active ? 'border-emerald-200 bg-emerald-50 text-emerald-700' : 'border-zinc-200 bg-zinc-50 text-zinc-700'">
                  {{ item.is_active ? t('admin.common.enabled') : t('admin.common.disabled') }}
                </span>
              </TableCell>
              <TableCell class="px-6 py-4 text-right">
                <div class="flex justify-end gap-2">
                  <Button size="sm" variant="outline" @click="toggleSuffix(item)">
                    {{ item.is_active ? t('admin.common.disabled') : t('admin.common.enabled') }}
                  </Button>
                  <Button size="sm" variant="outline" class="text-destructive" @click="removeSuffix(item)">{{ t('admin.common.delete') }}</Button>
                </div>
              </TableCell>
            </TableRow>
          </TableBody>
        </Table>
      </div>
    </div>
  </div>
</template>
