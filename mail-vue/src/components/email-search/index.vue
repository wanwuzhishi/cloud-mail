<template>
  <div ref="searchRef" class="email-search">
    <div class="search-placeholder" @click="openSearch">
      <Icon icon="iconoir:search" width="20" height="20"/>
      <span style="font-size: 15px">{{ t('searchMail') }}</span>
    </div>
    <Transition name="search-panel-fade" @after-enter="focusInput">
      <div v-if="visible" class="search-panel-shell" @click.stop>
        <el-card class="search-panel" shadow="always">
          <div class="search-input">
            <el-input
              ref="inputRef"
              v-model="searchText"
              autocomplete="off"
              :placeholder="searchPlaceholder"
              @keyup.enter="handleSearch"
            />
          </div>
          <div class="search-content">
            <div class="search-content-title">{{ t('searchType') }}</div>
            <div class="custom-style">
              <el-segmented
                v-model="searchType"
                :options="searchOptions"
                :props="{ label: 'label', value: 'type' }"
              />
            </div>
            <div class="search-date-range">
              <div class="search-date-field">
                <div class="search-date-label">{{ t('searchDateFrom') }}</div>
                <el-date-picker
                  v-model="startDate"
                  type="date"
                  value-format="YYYY-MM-DD"
                  :placeholder="t('searchStartDate')"
                />
              </div>
              <div class="search-date-field">
                <div class="search-date-label">{{ t('searchDateTo') }}</div>
                <el-date-picker
                  v-model="endDate"
                  type="date"
                  value-format="YYYY-MM-DD"
                  :placeholder="t('searchEndDate')"
                />
              </div>
            </div>
          </div>
          <div class="search-actions">
            <el-button @click="handleReset">{{ t('reset') }}</el-button>
            <el-button type="primary" @click="handleSearch">{{ t('search') }}</el-button>
          </div>
        </el-card>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import {computed, onBeforeUnmount, onMounted, ref, watch} from 'vue'
import {Icon} from '@iconify/vue'
import {useI18n} from 'vue-i18n'

const {t} = useI18n()
const emit = defineEmits(['search', 'reset'])
const props = defineProps({
  store: {
    type: Object,
    required: true,
  },
  searchOptions: {
    type: Array,
    required: true,
  },
})

const searchRef = ref(null)
const inputRef = ref(null)
const searchText = ref('')
const searchType = ref('subject')
const startDate = ref('')
const endDate = ref('')
const visible = ref(false)
const selectedSearchOption = computed(() =>
  props.searchOptions.find(option => option.type === searchType.value)
)
const searchPlaceholder = computed(() =>
  t('searchMailBy', {field: selectedSearchOption.value?.label || t('titleDesc')})
)

watch(
  () => props.searchOptions,
  options => {
    if (!options.some(option => option.type === searchType.value)) {
      searchType.value = options[0]?.type || 'subject'
    }
  },
  {immediate: true}
)

function openSearch() {
  if (visible.value) {
    closeSearch()
    return
  }

  restoreSavedSearch()
  visible.value = true
}

function focusInput() {
  inputRef.value?.focus()
}

function closeSearch() {
  visible.value = false
  restoreSavedSearch()
}

function restoreSavedSearch() {
  const params = props.store.searchParams
  searchText.value = params.text || ''
  searchType.value = props.searchOptions.some(option => option.type === params.type)
    ? params.type
    : props.searchOptions[0]?.type || 'subject'
  startDate.value = toLocalSearchDate(params.startDate)
  endDate.value = toLocalSearchDate(params.endDate)
}

function toLocalSearchDate(dateValue) {
  if (!dateValue) return ''

  const date = new Date(`${dateValue.replace(' ', 'T')}Z`)
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

function handleDocumentKeydown(event) {
  if (event.key === 'Escape' && visible.value) {
    closeSearch()
  }
}

function handleDocumentClick(event) {
  const target = event.target
  const isDatePickerPopup = target?.closest?.(
    '.el-picker__popper, .el-picker-panel'
  )

  if (
    !visible.value
    || searchRef.value?.contains(target)
    || isDatePickerPopup
  ) {
    return
  }

  event.preventDefault()
  event.stopPropagation()
  closeSearch()
}

function formatSearchDate(dateValue, endOfDay = false) {
  if (!dateValue) return ''

  const [year, month, day] = dateValue.split('-').map(Number)
  const localDate = new Date(
    year,
    month - 1,
    day,
    endOfDay ? 23 : 0,
    endOfDay ? 59 : 0,
    endOfDay ? 59 : 0,
  )

  return localDate.toISOString().slice(0, 19).replace('T', ' ')
}

function handleSearch() {
  props.store.setSearchParams({
    text: searchText.value.trim(),
    type: searchType.value,
    startDate: formatSearchDate(startDate.value),
    endDate: formatSearchDate(endDate.value, true),
  })
  emit('search', searchText.value.trim(), searchType.value)
  closeSearch()
}

function handleReset() {
  searchText.value = ''
  searchType.value = 'subject'
  startDate.value = ''
  endDate.value = ''
  props.store.setSearchParams({
    text: '',
    type: 'subject',
    startDate: '',
    endDate: '',
  })
  emit('reset')
  closeSearch()
}

onMounted(() => {
  document.addEventListener('click', handleDocumentClick, true)
  document.addEventListener('keydown', handleDocumentKeydown, true)
})

onBeforeUnmount(() => {
  document.removeEventListener('click', handleDocumentClick, true)
  document.removeEventListener('keydown', handleDocumentKeydown, true)
})
</script>

<style lang="scss" scoped>
.email-search {
  position: absolute;
  left: -1;
  top: 50%;
  transform: translateY(-50%);
  z-index: 2000;
  width: min(450px, calc(100% - 20px));
  height: 36px;
}

@media (min-width: 768px) {
  .email-search.account-show {
    left: 259px;
  }
}

.search-placeholder {
  width: 100%;
  height: 36px;
  border-radius: 6px;
  box-sizing: border-box;
}

.search-placeholder {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 10px;
  background: var(--el-fill-color-light);
  color: var(--el-text-color-secondary);
  text-align: left;
  white-space: nowrap;
  cursor: text;
  position: relative;
  z-index: 0;
}

.search-panel-shell {
  position: absolute;
  top: -4px;
  left: 0;
  z-index: 1001;
  width: 100%;
  background: var(--el-bg-color-overlay);
}

.search-panel {
  width: 100%;
  height: auto;
  min-height: 0;
  box-sizing: border-box;
  --el-card-bg-color: var(--el-bg-color-overlay);
  background-color: var(--el-card-bg-color) !important;
  border-radius: 6px !important;
  border: 1px solid var(--el-border-color) !important;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.16) !important;
  overflow: hidden;
  isolation: isolate;
}

.search-panel :deep(.el-card__body) {
  position: relative;
  z-index: 1;
  padding: 3px;
  background-color: var(--el-card-bg-color) !important;
}

.search-panel-shell {
  box-sizing: border-box;
}

.search-panel-fade-enter-active,
.search-panel-fade-leave-active {
  transition: opacity 0.2s, transform 0.2s;
}

.search-panel-fade-enter-from,
.search-panel-fade-leave-to {
  opacity: 0;
  transform: translateY(-4px);
}

.search-actions {
  display: flex;
  justify-content: flex-end;
  padding-top: 25px;
  padding-bottom: 15px;
  padding-right: 15px;
}

.search-content {
  padding: 20px 15px 0;
}

.search-content-title {
  font-size: 14px;
  font-weight: 700;
  line-height: 20px;
  color: var(--el-text-color-primary);
  width: 100%;
  text-align: left !important;
}

.custom-style {
  display: flex;
  justify-content: flex-start;
  margin-top: 8px;
}

.custom-style .el-segmented {
  --el-border-radius-base: 6px;
}

.search-date-range {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.search-date-field {
  flex: 1;
  min-width: 0;
}

.search-date-label {
  margin-bottom: 4px;
  font-size: 14px;
  font-weight: 700;
  line-height: 20px;
  color: var(--el-text-color-primary);
  text-align: left;
}

.search-date-field :deep(.el-date-editor) {
  width: 100%;
}

.search-input {
  --el-input-bg-color: var(--el-bg-color-overlay);
  margin: 4px -3px 0;
  padding: 0 8px 6px;
  border-bottom: solid 1px var(--el-border-color);
}

.search-input :deep(.el-input__wrapper),
.search-input :deep(.el-input__wrapper.is-focus) {
  box-shadow: none !important;
}

.search-input :deep(.el-input__inner::placeholder) {
  font-size: 15px;
}
</style>
