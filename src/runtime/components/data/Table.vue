<template>
  <div :class="ui.wrapper" v-bind="attrs">
    <table :class="[ui.base, ui.divide]">
      <slot v-if="$slots.caption || caption" name="caption">
        <caption :class="ui.caption">
          {{ caption }}
        </caption>
      </slot>
      <thead :class="ui.thead">
        <tr :class="ui.tr.base">
          <th v-if="modelValue" scope="col" :class="ui.checkbox.padding">
            <UCheckbox :model-value="indeterminate || selected.length === rows.length" :indeterminate="indeterminate" v-bind="ui.default.checkbox" aria-label="Select all" @change="onChange" />
          </th>

          <th v-if="$slots.expand" scope="col" :class="ui.tr.base">
            <span class="sr-only">Expand</span>
          </th>

          <th
            v-for="(column, index) in columns"
            :key="index"
            scope="col"
            :class="[ui.th.base, ui.th.padding, ui.th.color, ui.th.font, ui.th.size, column.class]"
            :aria-sort="getAriaSort(column)"
          >
            <slot :name="`${column.key}-header`" :column="column" :sort="sort" :on-sort="onSort">
              <UButton
                v-if="column.sortable"
                v-bind="{ ...(ui.default.sortButton || {}), ...sortButton }"
                :icon="(!sort.column || sort.column !== column.key) ? (sortButton.icon || ui.default.sortButton.icon) : sort.direction === 'asc' ? sortAscIcon : sortDescIcon"
                :label="column[columnAttribute]"
                @click="onSort(column)"
              />
              <span v-else>{{ column[columnAttribute] }}</span>
            </slot>
          </th>
        </tr>

        <tr v-if="loading && progress">
          <td :colspan="0" :class="ui.progress.wrapper">
            <UProgress v-bind="{ ...(ui.default.progress || {}), ...progress }" size="2xs" />
          </td>
        </tr>
      </thead>
      <tbody :class="ui.tbody">
        <tr v-if="loadingState && loading && !rows.length">
          <td :colspan="columns.length + (modelValue ? 1 : 0) + ($slots.expand ? 1 : 0)">
            <slot name="loading-state">
              <div :class="ui.loadingState.wrapper">
                <UIcon v-if="loadingState.icon" :name="loadingState.icon" :class="ui.loadingState.icon" aria-hidden="true" />
                <p :class="ui.loadingState.label">
                  {{ loadingState.label }}
                </p>
              </div>
            </slot>
          </td>
        </tr>

        <tr v-else-if="emptyState && !rows.length">
          <td :colspan="columns.length + (modelValue ? 1 : 0) + ($slots.expand ? 1 : 0)">
            <slot name="empty-state">
              <div :class="ui.emptyState.wrapper">
                <UIcon v-if="emptyState.icon" :name="emptyState.icon" :class="ui.emptyState.icon" aria-hidden="true" />
                <p :class="ui.emptyState.label">
                  {{ emptyState.label }}
                </p>
              </div>
            </slot>
          </td>
        </tr>

        <template v-else>
          <template v-for="(row, index) in rows" :key="index">
            <tr :class="[ui.tr.base, isSelected(row) && ui.tr.selected, $attrs.onSelect && ui.tr.active, row?.class]" @click="() => onSelect(row)">
              <td v-if="modelValue" :class="ui.checkbox.padding">
                <UCheckbox v-model="selected" :value="row" v-bind="ui.default.checkbox" aria-label="Select row" @click.capture.stop="() => onSelect(row)" />
              </td>

              <td
                v-if="$slots.expand"
                :class="[ui.td.base, ui.td.padding, ui.td.color, ui.td.font, ui.td.size]"
              >
                <UButton
                  v-bind="{ ...(ui.default.expandButton || {}), ...expandButton }"
                  :ui="{ icon: { base: [ui.expand.icon, openedRows.includes(index) && 'rotate-180'] } }"
                  @click="toggleOpened(index)"
                />
              </td>

              <td v-for="(column, subIndex) in columns" :key="subIndex" :class="[ui.td.base, ui.td.padding, ui.td.color, ui.td.font, ui.td.size, column?.rowClass, row[column.key]?.class]">
                <slot :name="`${column.key}-data`" :column="column" :row="row" :index="index" :get-row-data="(defaultValue) => getRowData(row, column.key, defaultValue)">
                  {{ getRowData(row, column.key) }}
                </slot>
              </td>
            </tr>
            <tr v-if="openedRows.includes(index)">
              <td colspan="100%">
                <slot
                  name="expand"
                  :row="row"
                  :index="index"
                />
              </td>
            </tr>
          </template>
        </template>
      </tbody>
    </table>
  </div>
</template>

<script lang="ts">
import { ref, computed, defineComponent, toRaw, toRef } from 'vue'
import type { PropType, AriaAttributes } from 'vue'
import { upperFirst } from 'scule'
import { defu } from 'defu'
import { useVModel } from '@vueuse/core'
import UIcon from '../elements/Icon.vue'
import UButton from '../elements/Button.vue'
import UProgress from '../elements/Progress.vue'
import UCheckbox from '../forms/Checkbox.vue'
import { useUI } from '../../composables/useUI'
import { mergeConfig, get } from '../../utils'
import type { Strategy, Button, ProgressColor, ProgressAnimation } from '../../types/index'
// @ts-expect-error
import appConfig from '#build/app.config'
import { table } from '#ui/ui.config'

const config = mergeConfig<typeof table>(appConfig.ui.strategy, appConfig.ui.table, table)

function defaultComparator<T> (a: T, z: T): boolean {
  return a === z
}

function defaultSort (a: any, b: any, direction: 'asc' | 'desc') {
  if (a === b) {
    return 0
  }

  if (direction === 'asc') {
    return a < b ? -1 : 1
  } else {
    return a > b ? -1 : 1
  }
}

interface Column {
  key: string
  sortable?: boolean
  sort?: (a: any, b: any, direction: 'asc' | 'desc') => number
  direction?: 'asc' | 'desc'
  class?: string
  rowClass?: string
  [key: string]: any
}

export default defineComponent({
  components: {
    UIcon,
    UButton,
    UProgress,
    UCheckbox
  },
  inheritAttrs: false,
  props: {
    modelValue: {
      type: Array,
      default: null
    },
    by: {
      type: [String, Function],
      default: () => defaultComparator
    },
    rows: {
      type: Array as PropType<{ [key: string]: any }[]>,
      default: () => []
    },
    columns: {
      type: Array as PropType<Column[]>,
      default: null
    },
    columnAttribute: {
      type: String,
      default: 'label'
    },
    sort: {
      type: Object as PropType<{ column: string, direction: 'asc' | 'desc' }>,
      default: () => ({})
    },
    sortMode: {
      type: String as PropType<'manual' | 'auto'>,
      default: 'auto'
    },
    sortButton: {
      type: Object as PropType<Button>,
      default: () => config.default.sortButton as Button
    },
    sortAscIcon: {
      type: String,
      default: () => config.default.sortAscIcon
    },
    sortDescIcon: {
      type: String,
      default: () => config.default.sortDescIcon
    },
    expandButton: {
      type: Object as PropType<Button>,
      default: () => config.default.expandButton as Button
    },
    loading: {
      type: Boolean,
      default: false
    },
    loadingState: {
      type: Object as PropType<{ icon: string, label: string }>,
      default: () => config.default.loadingState
    },
    emptyState: {
      type: Object as PropType<{ icon: string, label: string }>,
      default: () => config.default.emptyState
    },
    caption: {
      type: String,
      default: null
    },
    progress: {
      type: Object as PropType<{ color: ProgressColor, animation: ProgressAnimation }>,
      default: () => config.default.progress
    },
    class: {
      type: [String, Object, Array] as PropType<any>,
      default: () => ''
    },
    ui: {
      type: Object as PropType<Partial<typeof config> & { strategy?: Strategy }>,
      default: () => ({})
    }
  },
  emits: ['update:modelValue', 'update:sort'],
  setup (props, { emit, attrs: $attrs }) {
    const { ui, attrs } = useUI('table', toRef(props, 'ui'), config, toRef(props, 'class'))

    const columns = computed(() => props.columns ?? Object.keys(props.rows[0] ?? {}).map((key) => ({ key, label: upperFirst(key), sortable: false, class: undefined, sort: defaultSort }) as Column))

    const sort = useVModel(props, 'sort', emit, { passive: true, defaultValue: defu({}, props.sort, { column: null, direction: 'asc' }) })

    const openedRows = ref([])

    const savedSort = { column: sort.value.column, direction: null }

    const rows = computed(() => {
      if (!sort.value?.column || props.sortMode === 'manual') {
        return props.rows
      }

      const { column, direction } = sort.value

      return props.rows.slice().sort((a, b) => {
        const aValue = get(a, column)
        const bValue = get(b, column)

        const sort = columns.value.find((col) => col.key === column)?.sort ?? defaultSort

        return sort(aValue, bValue, direction)
      })
    })

    const selected = computed({
      get () {
        return props.modelValue
      },
      set (value) {
        emit('update:modelValue', value)
      }
    })

    const indeterminate = computed(() => selected.value && selected.value.length > 0 && selected.value.length < props.rows.length)

    const emptyState = computed(() => {
      if (props.emptyState === null) return null
      return { ...ui.value.default.emptyState, ...props.emptyState }
    })

    const loadingState = computed(() => {
      if (props.loadingState === null) return null
      return { ...ui.value.default.loadingState, ...props.loadingState }
    })

    function compare (a: any, z: any) {
      if (typeof props.by === 'string') {
        const property = props.by as unknown as any
        return a?.[property] === z?.[property]
      }
      return props.by(a, z)
    }

    function isSelected (row) {
      if (!props.modelValue) {
        return false
      }

      return selected.value.some((item) => compare(toRaw(item), toRaw(row)))
    }

    function onSort (column: { key: string, direction?: 'asc' | 'desc' }) {
      if (sort.value.column === column.key) {
        const direction = !column.direction || column.direction === 'asc' ? 'desc' : 'asc'

        if (sort.value.direction === direction) {
          sort.value = defu({}, savedSort, { column: null, direction: 'asc' })
        } else {
          sort.value = { column: sort.value.column, direction: sort.value.direction === 'asc' ? 'desc' : 'asc' }
        }
      } else {
        sort.value = { column: column.key, direction: column.direction || 'asc' }
      }
    }

    function onSelect (row) {
      if (!$attrs.onSelect) {
        return
      }

      // @ts-ignore
      $attrs.onSelect(row)
    }

    function selectAllRows () {
      // Create a new array to ensure reactivity
      const newSelected = [...selected.value]

      // If the row is not already selected, add it to the newSelected array
      props.rows.forEach((row) => {
        if (!isSelected(row)) {
          newSelected.push(row)
        }
      })

      // Reassign the array to trigger Vue's reactivity
      selected.value = newSelected
    }

    function onChange (checked: boolean) {
      if (checked) {
        selectAllRows()
      } else {
        selected.value = []
      }
    }

    function getRowData (row: Object, rowKey: string | string[], defaultValue: any = '') {
      return get(row, rowKey, defaultValue)
    }

    function toggleOpened (index: number) {
      if (openedRows.value.includes(index)) {
        openedRows.value = openedRows.value.filter((i) => i !== index)
      } else {
        openedRows.value.push(index)
      }
    }

    function getAriaSort (column: Column): AriaAttributes['aria-sort'] {
      if (!column.sortable) {
        return undefined
      }

      if (sort.value.column !== column.key) {
        return 'none'
      }

      if (sort.value.direction === 'asc') {
        return 'ascending'
      }

      if (sort.value.direction === 'desc') {
        return 'descending'
      }

      return undefined
    }

    return {
      // eslint-disable-next-line vue/no-dupe-keys
      ui,
      attrs,
      // eslint-disable-next-line vue/no-dupe-keys
      sort,
      // eslint-disable-next-line vue/no-dupe-keys
      columns,
      // eslint-disable-next-line vue/no-dupe-keys
      rows,
      selected,
      indeterminate,
      // eslint-disable-next-line vue/no-dupe-keys
      emptyState,
      // eslint-disable-next-line vue/no-dupe-keys
      loadingState,
      openedRows,
      isSelected,
      onSort,
      onSelect,
      onChange,
      getRowData,
      toggleOpened,
      getAriaSort
    }
  }
})
</script>
