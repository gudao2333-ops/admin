<script setup lang="ts">
import { onMounted } from 'vue'
import { useI18n } from 'vue-i18n'
import { useTelegramBotSettings } from '@/composables/useTelegramBotSettings'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { ArrowDown, ArrowUp, Loader2, Plus, Save, Trash2 } from 'lucide-vue-next'

const { t } = useI18n()
const {
  addMenuItem,
  currentLang,
  fetchConfig,
  form,
  languages,
  loading,
  menuActionTypes,
  moveMenuItem,
  removeMenuItem,
  saveConfig,
  saving,
} = useTelegramBotSettings()

onMounted(() => {
  fetchConfig()
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex flex-wrap items-center justify-between gap-4">
      <div>
        <h2 class="text-2xl font-bold tracking-tight">{{ t('telegramBot.settings.menuTitle') }}</h2>
        <p class="text-muted-foreground">{{ t('telegramBot.settings.menuDesc') }}</p>
      </div>
      <div class="flex flex-wrap items-center gap-3">
        <div class="flex rounded-lg border border-border bg-card p-1">
          <button
            v-for="lang in languages"
            :key="lang.code"
            class="rounded-md px-3 py-1.5 text-xs font-medium transition-colors"
            :class="currentLang === lang.code ? 'bg-secondary text-foreground' : 'text-muted-foreground hover:text-foreground'"
            @click="currentLang = lang.code"
          >
            {{ lang.name }}
          </button>
        </div>
        <Button :disabled="saving || loading" @click="saveConfig">
          <Loader2 v-if="saving" class="mr-2 h-4 w-4 animate-spin" />
          <Save v-else class="mr-2 h-4 w-4" />
          {{ t('telegramBot.settings.save') }}
        </Button>
      </div>
    </div>

    <Card>
      <CardHeader>
        <div class="flex items-center justify-between">
          <div>
            <CardTitle>{{ t('telegramBot.settings.menuTitle') }}</CardTitle>
            <CardDescription>{{ t('telegramBot.settings.menuDesc') }}</CardDescription>
          </div>
          <div class="flex items-center gap-2">
            <span class="rounded bg-muted px-2 py-1 text-xs text-muted-foreground">{{ currentLang }}</span>
            <Button type="button" size="sm" variant="outline" @click="addMenuItem">
              <Plus class="mr-1 h-4 w-4" />
              {{ t('telegramBot.settings.menuAdd') }}
            </Button>
          </div>
        </div>
      </CardHeader>
      <CardContent class="space-y-3">
        <div v-if="form.menu.items.length === 0" class="rounded-lg border border-dashed border-border p-6 text-center text-sm text-muted-foreground">
          {{ t('telegramBot.settings.menuEmpty') }}
        </div>
        <div
          v-for="(item, index) in form.menu.items"
          :key="index"
          class="space-y-3 rounded-lg border border-border p-4"
        >
          <div class="flex flex-wrap items-center gap-2">
            <input
              :id="`menu-enabled-${index}`"
              v-model="item.enabled"
              type="checkbox"
              class="h-4 w-4 accent-primary"
            />
            <Input
              v-model="item.key"
              :placeholder="t('telegramBot.settings.menuKeyPlaceholder')"
              class="min-w-[120px] flex-1"
            />
            <Button
              type="button"
              size="icon"
              variant="ghost"
              :disabled="index === 0"
              @click="moveMenuItem(index, 'up')"
            >
              <ArrowUp class="h-4 w-4" />
            </Button>
            <Button
              type="button"
              size="icon"
              variant="ghost"
              :disabled="index === form.menu.items.length - 1"
              @click="moveMenuItem(index, 'down')"
            >
              <ArrowDown class="h-4 w-4" />
            </Button>
            <Button type="button" size="icon" variant="ghost" @click="removeMenuItem(index)">
              <Trash2 class="h-4 w-4 text-destructive" />
            </Button>
          </div>
          <div class="space-y-1">
            <Label>{{ t('telegramBot.settings.menuLabel') }}</Label>
            <Input v-model="item.label[currentLang]" :placeholder="t('telegramBot.settings.menuLabelPlaceholder')" />
          </div>
          <div class="grid grid-cols-1 gap-3 md:grid-cols-3">
            <div class="space-y-1">
              <Label>{{ t('telegramBot.settings.menuActionType') }}</Label>
              <Select v-model="item.action.type">
                <SelectTrigger>
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem v-for="actionType in menuActionTypes" :key="actionType" :value="actionType">
                    {{ t(`telegramBot.settings.menuActionType_${actionType}`) }}
                  </SelectItem>
                </SelectContent>
              </Select>
            </div>
            <div class="space-y-1">
              <Label>{{ t('telegramBot.settings.menuActionValue') }}</Label>
              <Input v-model="item.action.value" :placeholder="t('telegramBot.settings.menuActionValuePlaceholder')" />
            </div>
            <div class="space-y-1">
              <Label>{{ t('telegramBot.settings.menuOrder') }}</Label>
              <Input v-model.number="item.order" type="number" />
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  </div>
</template>
