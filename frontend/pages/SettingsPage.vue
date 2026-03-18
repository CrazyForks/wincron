<script setup>
import { onMounted } from "vue"
import { storeToRefs } from "pinia"
import { useCronStore } from "../stores/cron.js"
import { useI18n } from "vue-i18n"
import { btn, btnDanger, btnPrimary } from "../ui/buttonClasses.js"
import AppScrollbar from "../components/AppScrollbar.vue"

import SettingsExportDialog from "../components/SettingsExportDialog.vue"
import SettingsImportDialog from "../components/SettingsImportDialog.vue"
import SettingsShortcutDialog from "../components/SettingsShortcutDialog.vue"

const cron = useCronStore()
const { error, silentStart, lightweightMode, autoStart, runInTray } = storeToRefs(cron)

const { t } = useI18n()

onMounted(async () => {
  await cron.loadSettings()
})

async function openDataDir() {
  await cron.openDataDir()
}

async function openEnvironmentVariables() {
  await cron.openEnvironmentVariables()
}

const onRunInTrayChange = (ev) => cron.setRunInTray(!!ev?.target?.checked)
const onSilentStartChange = (ev) => cron.setSilentStart(!!ev?.target?.checked)
const onLightweightModeChange = (ev) => cron.setLightweightMode(!!ev?.target?.checked)
const onAutoStartChange = (ev) => cron.setAutoStart(!!ev?.target?.checked)
</script>

<template>
  <AppScrollbar root-class="h-full" view-class="mx-auto max-w-[920px] p-2 sm:p-3 md:p-5">
    <section class="rounded-2xl border border-slate-200 bg-white p-2.5 shadow-[0_10px_30px_rgba(2,6,23,0.08)] sm:p-3.5">
      <div class="flex items-start justify-between gap-2 px-2 pt-2 pb-1.5 sm:gap-3 sm:px-3 sm:pt-3 sm:pb-2">
        <div>
          <h2 class="text-sm sm:text-base">{{ $t("settings.title") }}</h2>
          <div class="mt-0.5 text-[10px] text-slate-500 sm:text-xs">{{ $t("settings.subtitle") }}</div>
        </div>
      </div>

      <div v-if="error" class="mx-2 mb-2 rounded-xl border border-red-600/25 bg-red-50 px-2.5 py-2 text-xs text-red-800 sm:mx-3 sm:mb-3 sm:px-3 sm:py-2.5 sm:text-sm">
        {{ error }}
      </div>

      <div class="px-2 pb-3 space-y-2.5 sm:px-3 sm:pb-3.5 sm:space-y-3">
        <div class="rounded-2xl border border-slate-200 bg-slate-50 p-2.5 sm:p-3">
          <div class="text-xs font-semibold text-slate-900 sm:text-sm">{{ $t("settings.startup") }}</div>
          <div class="mt-1.5 space-y-2 sm:mt-2 sm:space-y-3">
            <div class="flex items-start justify-between gap-2 sm:gap-4">
              <div>
                <div class="text-xs text-slate-900 sm:text-sm">{{ $t("settings.run_on_boot") }}</div>
                <div class="mt-0.5 text-xs text-slate-500">{{ $t("settings.run_on_boot_help") }}</div>
              </div>
              <input type="checkbox" v-model="autoStart" class="mt-0.5" @change="onAutoStartChange" />
            </div>

            <div class="flex items-start justify-between gap-2 sm:gap-4">
              <div>
                <div class="text-xs text-slate-900 sm:text-sm">{{ $t("settings.silent_start") }}</div>
                <div class="mt-0.5 text-xs text-slate-500">{{ $t("settings.silent_start_help") }}</div>
              </div>
              <input type="checkbox" v-model="silentStart" class="mt-0.5" @change="onSilentStartChange" />
            </div>

            <div class="flex items-start justify-between gap-2 sm:gap-4">
              <div>
                <div class="text-xs text-slate-900 sm:text-sm">{{ $t("settings.lightweight_mode") }}</div>
                <div class="mt-0.5 text-xs text-slate-500">{{ $t("settings.lightweight_mode_help") }}</div>
              </div>
              <input type="checkbox" v-model="lightweightMode" class="mt-0.5" @change="onLightweightModeChange" />
            </div>

            <div class="flex items-start justify-between gap-2 sm:gap-4">
              <div>
                <div class="text-xs text-slate-900 sm:text-sm">{{ $t("settings.run_in_tray") }}</div>
                <div class="mt-0.5 text-xs text-slate-500">{{ $t("settings.run_in_tray_help") }}</div>
              </div>
              <input type="checkbox" v-model="runInTray" class="mt-0.5" @change="onRunInTrayChange" />
            </div>
          </div>
        </div>

        <div class="rounded-2xl border border-slate-200 bg-slate-50 p-2.5 sm:p-3">
          <div class="text-xs font-semibold text-slate-900 sm:text-sm">{{ $t("settings.data_management") }}</div>
          <div class="mt-1.5 flex flex-wrap gap-2 sm:mt-2 sm:gap-2.5">
            <SettingsExportDialog :btn="btn" :btnPrimary="btnPrimary" />
            <SettingsImportDialog :btn="btn" :btnPrimary="btnPrimary" />
            <SettingsShortcutDialog :btn="btn" :btnPrimary="btnPrimary" :btnDanger="btnDanger" />
          </div>
        </div>

        <div class="rounded-2xl border border-slate-200 bg-slate-50 p-2.5 sm:p-3">
          <div class="text-xs font-semibold text-slate-900 sm:text-sm">{{ $t("settings.quick_access") }}</div>
          <div class="mt-1.5 flex flex-wrap gap-2 sm:mt-2 sm:gap-2.5">
            <button :class="btn" @click="openDataDir">{{ $t("settings.open_data_dir") }}</button>
            <button :class="btn" @click="openEnvironmentVariables">{{ $t("settings.open_environment_variables") }}</button>
          </div>
        </div>

      </div>
    </section>
  </AppScrollbar>
</template>
