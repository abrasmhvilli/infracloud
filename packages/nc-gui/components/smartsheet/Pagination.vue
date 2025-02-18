<script setup lang="ts">
import type { PaginatedType } from 'nocodb-sdk'
import SidebarIcon from '~icons/nc-icons/sidebar'
import { IsGroupByInj, computed, iconMap, inject, isRtlLang, useI18n } from '#imports'
import type { Language } from '#imports'

interface Props {
  paginationData: PaginatedType
  changePage: (page: number) => void
  alignCountOnRight?: boolean
  hidePagination?: boolean
  hideSidebars?: boolean
  customLabel?: string
  fixedSize?: number
  extraStyle?: string
}

const props = defineProps<Props>()

const emits = defineEmits(['update:paginationData'])

const { locale } = useI18n()

const vPaginationData = useVModel(props, 'paginationData', emits)

const { alignCountOnRight, customLabel, changePage } = props

const fixedSize = toRef(props, 'fixedSize')

const extraStyle = toRef(props, 'extraStyle')

const isPublic = inject(IsPublicInj, ref(false))

const isGroupBy = inject(IsGroupByInj, ref(false))

const { isLeftSidebarOpen, isRightSidebarOpen } = storeToRefs(useSidebarStore())

const { isPaginationLoading } = storeToRefs(useViewsStore())

const count = computed(() => vPaginationData.value?.totalRows ?? Infinity)

const size = computed(() => vPaginationData.value?.pageSize ?? 25)

const page = computed({
  get: () => vPaginationData?.value?.page ?? 1,
  set: async (p) => {
    isPaginationLoading.value = true
    try {
      await changePage?.(p)
    } catch (e) {
      console.error(e)
    } finally {
      isPaginationLoading.value = false
    }
  },
})

const isRTLLanguage = computed(() => isRtlLang(locale.value as keyof typeof Language))
</script>

<template>
  <div
    class="flex items-center bg-white border-gray-200 h-10 nc-pagination-wrapper"
    :class="{ 'border-t-1': !isGroupBy }"
    :style="`${fixedSize ? `width: ${fixedSize}px;` : ''}${
      isGroupBy ? 'margin-top:1px; border-radius: 0 0 12px 12px !important;' : ''
    }${extraStyle}`"
  >
    <NcTooltip v-if="!isPublic && hideSidebars !== true" class="ml-2" placement="topLeft" hide-on-click>
      <template #title>
        {{
          isLeftSidebarOpen
            ? `${$t('general.hide')} ${$t('objects.sidebar').toLowerCase()}`
            : `${$t('general.show')} ${$t('objects.sidebar').toLowerCase()}`
        }}
      </template>
      <div
        class="nc-sidebar-left-toggle-icon hover:after:(bg-primary bg-opacity-75) hover:(bg-gray-100 border-gray-200) border-gray-200 group flex items-center justify-center rounded-md h-full px-1.75 h-7 cursor-pointer text-gray-500 hover:text-gray-700"
        :class="{
          'bg-gray-100': !isLeftSidebarOpen,
        }"
        @click="isLeftSidebarOpen = !isLeftSidebarOpen"
      >
        <SidebarIcon class="cursor-pointer transform transition-transform duration-500 rounded-md rotate-180" />
      </div>
    </NcTooltip>
    <slot name="add-record" />
    <div class="flex-1">
      <span
        v-if="!alignCountOnRight && count !== null && count !== Infinity"
        class="caption ml-2.5 text-gray-500 text-xs"
        data-testid="grid-pagination"
      >
        {{ count }} {{ customLabel ? customLabel : count !== 1 ? $t('objects.records') : $t('objects.record') }}
      </span>
    </div>

    <template v-if="!hidePagination">
      <a-pagination
        v-if="count !== Infinity"
        v-model:current="page"
        v-model:page-size="size"
        size="small"
        class="!text-xs !m-1 nc-pagination"
        :class="{ 'rtl-pagination': isRTLLanguage }"
        :total="+count"
        show-less-items
        :show-size-changer="false"
      />
      <div v-else class="mx-auto flex items-center mt-n1" style="max-width: 250px">
        <span class="text-xs" style="white-space: nowrap"> Change page:</span>
        <a-input :value="page" size="small" class="ml-1 !text-xs" type="number" @keydown.enter="changePage(page)">
          <template #suffix>
            <component :is="iconMap.returnKey" class="mt-1" @click="changePage(page)" />
          </template>
        </a-input>
      </div>
    </template>

    <div class="flex-1 text-right pr-2">
      <span
        v-if="alignCountOnRight && count !== null && count !== Infinity"
        class="caption nc-grid-row-count mr-2.5 text-gray-500 text-xs"
        data-testid="grid-pagination"
      >
        {{ count }} {{ customLabel ? customLabel : count !== 1 ? $t('objects.records') : $t('objects.record') }}
      </span>
    </div>

    <NcTooltip v-if="!isPublic && hideSidebars !== true" placement="topRight" hide-on-click>
      <template #title>
        {{
          isRightSidebarOpen
            ? `${$t('general.hide')} ${$t('objects.sidebar').toLowerCase()}`
            : `${$t('general.show')} ${$t('objects.sidebar').toLowerCase()}`
        }}
      </template>

      <div
        class="flex flex-row items-center justify-center !rounded-md !p-1.75 border-gray-100 cursor-pointer bg-white hover:bg-gray-100 text-gray-500 hover:text-gray-700 nc-sidebar-right-toggle-icon"
        :class="{
          '!bg-gray-100': !isRightSidebarOpen,
        }"
        type="ghost"
        @click="isRightSidebarOpen = !isRightSidebarOpen"
      >
        <SidebarIcon class="w-4 h-4" />
      </div>
    </NcTooltip>
    <div class="w-2"></div>
  </div>
</template>

<style lang="scss">
.nc-pagination-wrapper {
  .ant-pagination-item-active {
    a {
      @apply text-sm !text-gray-700 !hover:text-gray-800;
    }
  }
}
</style>

<style scoped>
:deep(.ant-pagination-item a) {
  @apply text-sm !leading-[21px] !no-underline;
}

:deep(.nc-pagination .ant-pagination-item) {
  @apply !border-0 !pt-0.25;
}

:deep(.ant-pagination-item:not(.ant-pagination-item-active) a) {
  line-height: 21px !important;
  @apply text-sm !text-gray-400;
}

:deep(.ant-pagination-item-link) {
  @apply text-gray-800 flex items-center justify-center;
}

:deep(.ant-pagination-item.ant-pagination-item-active) {
  @apply !bg-transparent;
}

:deep(.rtl-pagination .ant-pagination-prev .ant-pagination-item-link),
:deep(.rtl-pagination .ant-pagination-next .ant-pagination-item-link) {
  @apply transform rotate-180;
}
</style>
