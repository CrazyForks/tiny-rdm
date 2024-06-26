<script setup>
import { computed, h, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import AddLink from '@/components/icons/AddLink.vue'
import { NButton, NIcon, useThemeVars } from 'naive-ui'
import { types, types as redisTypes } from '@/consts/support_redis_type.js'
import EditableTableColumn from '@/components/common/EditableTableColumn.vue'
import useDialogStore from 'stores/dialog.js'
import { includes, isEmpty, size } from 'lodash'
import { decodeTypes, formatTypes } from '@/consts/value_view_type.js'
import useBrowserStore from 'stores/browser.js'
import LoadList from '@/components/icons/LoadList.vue'
import LoadAll from '@/components/icons/LoadAll.vue'
import IconButton from '@/components/common/IconButton.vue'
import ContentSearchInput from '@/components/content_value/ContentSearchInput.vue'
import { formatBytes } from '@/utils/byte_convert.js'
import copy from 'copy-text-to-clipboard'

const i18n = useI18n()
const themeVars = useThemeVars()

const props = defineProps({
    name: String,
    db: Number,
    keyPath: String,
    keyCode: {
        type: Array,
        default: null,
    },
    ttl: {
        type: Number,
        default: -1,
    },
    value: {
        type: Array,
        default: () => [],
    },
    size: Number,
    length: Number,
    viewAs: {
        type: String,
        default: formatTypes.RAW,
    },
    decode: {
        type: String,
        default: decodeTypes.NONE,
    },
    end: Boolean,
    loading: Boolean,
})

const emit = defineEmits(['loadmore', 'loadall', 'match'])

/**
 *
 * @type {ComputedRef<string|number[]>}
 */
const keyName = computed(() => {
    return !isEmpty(props.keyCode) ? props.keyCode : props.keyPath
})
const filterType = ref(1)

const browserStore = useBrowserStore()
const dialogStore = useDialogStore()
const keyType = redisTypes.STREAM
const idColumn = computed(() => ({
    key: 'id',
    title: 'ID',
    align: 'center',
    titleAlign: 'center',
    resizable: true,
}))

const valueFilterOption = ref(null)
const valueColumn = computed(() => ({
    key: 'value',
    title: () => i18n.t('common.value'),
    align: 'left',
    titleAlign: 'center',
    resizable: true,
    filterOptionValue: valueFilterOption.value,
    filter: (value, row) => {
        const v = value.toString()
        if (row.dv) {
            return includes(row.dv, v)
        }
        for (const k in row.v) {
            if (includes(k, v) || includes(row.v[k], v)) {
                return true
            }
        }
        return false
    },
    // sorter: (row1, row2) => row1.value - row2.value,
    render: (row) => {
        return h('pre', { class: 'pre-wrap' }, row.dv)
    },
}))
const actionColumn = {
    key: 'action',
    title: () => i18n.t('interface.action'),
    width: 80,
    align: 'center',
    titleAlign: 'center',
    fixed: 'right',
    render: (row) => {
        return h(EditableTableColumn, {
            bindKey: row.id,
            readonly: true,
            onCopy: async () => {
                copy(JSON.stringify(row.v))
                $message.success(i18n.t('interface.copy_succ'))
            },
            onDelete: async () => {
                try {
                    const { success, msg } = await browserStore.removeStreamValues({
                        server: props.name,
                        db: props.db,
                        key: keyName.value,
                        ids: row.id,
                    })
                    if (success) {
                        $message.success(i18n.t('dialogue.delete.success', { key: row.id }))
                    } else {
                        $message.error(msg)
                    }
                } catch (e) {
                    $message.error(e.message)
                }
            },
        })
    },
}
const columns = computed(() => [idColumn.value, valueColumn.value, actionColumn])

const entries = computed(() => {
    const len = size(props.value)
    return `${len} / ${Math.max(len, props.length)}`
})

const loadProgress = computed(() => {
    const len = size(props.value)
    return (len * 100) / Math.max(len, props.length)
})

const showMemoryUsage = computed(() => {
    return !isNaN(props.size) && props.size > 0
})

const onAddRow = () => {
    dialogStore.openAddFieldsDialog(props.name, props.db, props.keyPath, props.keyCode, types.STREAM)
}

const onFilterInput = (val) => {
    valueFilterOption.value = val
}

const onMatchInput = (matchVal, filterVal) => {
    valueFilterOption.value = filterVal
    emit('match', matchVal)
}

const onUpdateFilter = (filters, sourceColumn) => {
    valueFilterOption.value = filters[sourceColumn.key]
}

const searchInputRef = ref(null)
defineExpose({
    reset: () => {
        searchInputRef.value?.reset()
    },
})
</script>

<template>
    <div class="content-wrapper flex-box-v">
        <slot name="toolbar" />
        <div class="tb2 value-item-part flex-box-h">
            <div class="flex-box-h">
                <content-search-input
                    ref="searchInputRef"
                    @filter-changed="onFilterInput"
                    @match-changed="onMatchInput" />
            </div>
            <div class="flex-item-expand"></div>
            <n-button-group>
                <icon-button
                    :disabled="props.end || props.loading"
                    :icon="LoadList"
                    border
                    size="18"
                    t-tooltip="interface.load_more_entries"
                    @click="emit('loadmore')" />
                <icon-button
                    :disabled="props.end || props.loading"
                    :icon="LoadAll"
                    border
                    size="18"
                    t-tooltip="interface.load_all_entries"
                    @click="emit('loadall')" />
            </n-button-group>
            <n-button :focusable="false" plain @click="onAddRow">
                <template #icon>
                    <n-icon :component="AddLink" size="18" />
                </template>
                {{ $t('interface.add_row') }}
            </n-button>
        </div>
        <!-- loaded progress -->
        <n-progress
            :border-radius="0"
            :color="props.end ? '#0000' : themeVars.primaryColor"
            :height="2"
            :percentage="loadProgress"
            :processing="props.loading"
            :show-indicator="false"
            status="success"
            type="line" />
        <div class="value-wrapper value-item-part flex-box-v flex-item-expand">
            <n-data-table
                :bordered="false"
                :bottom-bordered="false"
                :columns="columns"
                :data="props.value"
                :loading="props.loading"
                :row-key="(row) => row.id"
                :single-column="true"
                :single-line="false"
                class="flex-item-expand"
                flex-height
                size="small"
                striped
                virtual-scroll
                @update:filters="onUpdateFilter" />
        </div>

        <div class="value-footer flex-box-h">
            <n-text v-if="!isNaN(props.length)">{{ $t('interface.entries') }}: {{ entries }}</n-text>
            <n-divider v-if="showMemoryUsage" vertical />
            <n-text v-if="showMemoryUsage">{{ $t('interface.memory_usage') }}: {{ formatBytes(props.size) }}</n-text>
            <div class="flex-item-expand"></div>
        </div>
    </div>
</template>

<style lang="scss" scoped>
.value-footer {
    border-top: v-bind('themeVars.borderColor') 1px solid;
    background-color: v-bind('themeVars.tableHeaderColor');
}
</style>
