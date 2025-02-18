<script setup lang="ts">
import { type ColumnType, UITypes } from 'nocodb-sdk'
import MdiFormatListGroup from '~icons/mdi/format-list-group'
import {
  ActiveViewInj,
  IsLockedInj,
  MetaInj,
  computed,
  getSortDirectionOptions,
  inject,
  ref,
  useMenuCloseOnEsc,
  useSmartsheetStoreOrThrow,
} from '#imports'

const groupingUidt = [
  UITypes.SingleSelect,
  UITypes.MultiSelect,
  UITypes.Checkbox,
  UITypes.Date,
  UITypes.SingleLineText,
  UITypes.Number,
]

const meta = inject(MetaInj, ref())
const view = inject(ActiveViewInj, ref())
const isLocked = inject(IsLockedInj, ref(false))

const { gridViewCols, updateGridViewColumn } = useGridViewColumnOrThrow()

const _groupBy = ref<{ fk_column_id?: string; sort: string; order: number }[]>([])

const groupBy = computed<{ fk_column_id?: string; sort: string; order: number }[]>(() => {
  const tempGroupBy: { fk_column_id?: string; sort: string; order: number }[] = []
  Object.values(gridViewCols.value).forEach((col) => {
    if (col.group_by) {
      tempGroupBy.push({
        fk_column_id: col.fk_column_id,
        sort: col.group_by_sort || 'asc',
        order: col.group_by_order || 1,
      })
    }
  })
  tempGroupBy.sort((a, b) => a.order - b.order)
  return tempGroupBy
})

const groupedByColumnIds = computed(() => groupBy.value.map((g) => g.fk_column_id).filter((g) => g))

const { eventBus } = useSmartsheetStoreOrThrow()

const { isMobileMode } = useGlobal()

const fieldsToGroupBy = computed(() => {
  const fields = meta.value?.columns || []

  return fields.filter((field) => {
    return groupingUidt.includes(field.uidt as UITypes)
  })
})

const columns = computed(() => meta.value?.columns || [])

const columnByID = computed(() =>
  columns.value.reduce((obj, col) => {
    obj[col.id!] = col

    return obj
  }, {} as Record<string, ColumnType>),
)

const getColumnUidtByID = (key?: string) => {
  if (!key) return ''
  return columnByID.value[key]?.uidt || ''
}

const open = ref(false)

useMenuCloseOnEsc(open)

const saveGroupBy = async () => {
  if (view.value?.id) {
    try {
      for (const gby of _groupBy.value) {
        if (!gby.fk_column_id) continue
        const col = gridViewCols.value[gby.fk_column_id]
        if (col && (!col.group_by || col.group_by_order !== gby.order || col.group_by_sort !== gby.sort)) {
          await updateGridViewColumn(gby.fk_column_id, {
            group_by: true,
            group_by_order: gby.order,
            group_by_sort: gby.sort,
          })
        }
      }

      for (const gby of groupBy.value) {
        if (!gby.fk_column_id) continue
        if (_groupBy.value.find((g) => g.fk_column_id === gby.fk_column_id)) continue
        const col = gridViewCols.value[gby.fk_column_id]
        if (col && col.group_by) {
          await updateGridViewColumn(gby.fk_column_id, {
            group_by: false,
            group_by_order: undefined,
            group_by_sort: undefined,
          })
        }
      }

      eventBus.emit(SmartsheetStoreEvents.GROUP_BY_RELOAD)
    } catch (e) {
      message.error('There was an error while updating view!')
    }
  } else {
    message.error('View not found!!!')
  }
}

const addFieldToGroupBy = async () => {
  _groupBy.value.push({ fk_column_id: undefined, sort: 'asc', order: _groupBy.value.length + 1 })
}

const removeFieldFromGroupBy = async (index: string | number) => {
  if (groupedByColumnIds.value.length === 0) {
    open.value = false
    return
  }
  _groupBy.value.splice(+index, 1)
  await saveGroupBy()
  if (_groupBy.value.length === 0) {
    addFieldToGroupBy()
  }
}

watch(open, () => {
  if (open.value) {
    _groupBy.value = [...groupBy.value]
    if (_groupBy.value.length === 0) {
      addFieldToGroupBy()
    }
  }
})
</script>

<template>
  <NcDropdown v-model:visible="open" offset-y class="" :trigger="['click']" overlay-class-name="nc-dropdown-group-by-menu">
    <div :class="{ 'nc-badge nc-active-btn': groupedByColumnIds?.length }">
      <a-button v-e="['c:group-by']" class="nc-group-by-menu-btn nc-toolbar-btn" :disabled="isLocked">
        <div class="flex items-center gap-2">
          <MdiFormatListGroup class="h-4 w-4" />

          <!-- Group By -->
          <span v-if="!isMobileMode" class="text-capitalize !text-sm font-medium">{{ $t('activity.groupBy') }}</span>

          <span v-if="groupedByColumnIds?.length" class="nc-count-badge">{{ groupedByColumnIds.length }}</span>
        </div>
      </a-button>
    </div>
    <template #overlay>
      <div
        :class="{ ' min-w-[400px]': _groupBy.length }"
        class="flex flex-col bg-white shadow-lg rounded-md overflow-auto border-1 border-gray-50 menu-filter-dropdown max-h-[max(80vh,500px)] py-6 pl-6"
        data-testid="nc-group-by-menu"
      >
        <div class="group-by-grid pb-1 mb-2 max-h-100 nc-scrollbar-md pr-5" @click.stop>
          <template
            v-for="[i, group] of Object.entries(_groupBy)"
            :key="`grouped-by-${group.fk_column_id}`"
            class="nc-group-menu"
          >
            <LazySmartsheetToolbarFieldListAutoCompleteDropdown
              v-model="group.fk_column_id"
              class="caption nc-sort-field-select"
              :columns="
                fieldsToGroupBy.filter((f) => (f.id && !groupedByColumnIds.includes(f.id)) || f.id === group.fk_column_id)
              "
              :allow-empty="true"
              @change="saveGroupBy"
              @click.stop
            />
            <NcSelect
              ref=""
              v-model:value="group.sort"
              class="shrink grow-0 nc-sort-dir-select"
              :label="$t('labels.operation')"
              dropdown-class-name="sort-dir-dropdown nc-dropdown-sort-dir"
              :disabled="!group.fk_column_id"
              @change="saveGroupBy"
              @click.stop
            >
              <a-select-option
                v-for="(option, j) of getSortDirectionOptions(getColumnUidtByID(group.fk_column_id))"
                :key="j"
                :value="option.value"
              >
                <span>{{ option.text }}</span>
              </a-select-option>
            </NcSelect>

            <a-tooltip placement="right" title="Remove">
              <NcButton class="nc-group-by-item-remove-btn" size="small" type="text" @click.stop="removeFieldFromGroupBy(i)">
                <GeneralIcon icon="delete" class="" />
              </NcButton>
            </a-tooltip>
          </template>
        </div>
        <NcButton
          v-if="fieldsToGroupBy.length > _groupBy.length && _groupBy.length < 3"
          class="nc-add-group-btn"
          style="width: fit-content"
          size="small"
          type="text"
          :disabled="groupedByColumnIds.length < _groupBy.length"
          @click.stop="addFieldToGroupBy()"
        >
          <div class="flex gap-1 items-center" :class="{ 'text-brand-500': groupedByColumnIds.length >= _groupBy.length }">
            <div class="flex">
              {{ $t('activity.addSubGroup') }}
            </div>
            <GeneralIcon icon="plus" />
          </div>
        </NcButton>
      </div>
    </template>
  </NcDropdown>
</template>

<style scoped>
.group-by-grid {
  display: grid;
  grid-template-columns: auto 150px 22px;
  @apply gap-[12px];
}
</style>
