<template>
  <Dialog v-model="show" :options="{ title: __('Bulk Edit') }">
    <template #body-content>
      <div class="mb-4">
        <div class="mb-1.5 text-sm text-ink-gray-5">{{ __('Field') }}</div>
        <Autocomplete
          :value="field.label"
          :options="fields.data"
          @change="(e) => changeField(e)"
          :placeholder="__('Source')"
        />
      </div>
      <div>
        <div class="mb-1.5 text-sm text-ink-gray-5">{{ __('Value') }}</div>
        <component
          :is="getValueComponent(field)"
          :value="newValue"
          size="md"
          @change="(v) => updateValue(v)"
          :placeholder="__('Contact Us')"
        />
      </div>
    </template>
    <template #actions>
      <Button
        class="w-full"
        variant="solid"
        @click="updateValues"
        :loading="loading"
        :label="__('Update {0} Records', [recordCount])"
      />
    </template>
  </Dialog>
</template>

<script setup>
import Link from '@/components/Controls/Link.vue'
import Autocomplete from '@/components/frappe-ui/Autocomplete.vue'
import { capture } from '@/telemetry'
import {
  FormControl,
  call,
  createResource,
  TextEditor,
  DatePicker,
} from 'frappe-ui'
import { ref, computed, onMounted, h } from 'vue'

const typeCheck = ['Check']
const typeLink = ['Link', 'Dynamic Link']
const typeNumber = ['Float', 'Int', 'Currency', 'Percent']
const typeSelect = ['Select']
const typeEditor = ['Text Editor']
const typeDate = ['Date', 'Datetime']

const props = defineProps({
  doctype: {
    type: String,
    required: true,
  },
  selectedValues: {
    type: Set,
    required: true,
  },
})

const show = defineModel()

const emit = defineEmits(['reload'])

const fields = createResource({
  url: 'crm.api.doc.get_fields',
  cache: ['fields', props.doctype],
  params: {
    doctype: props.doctype,
  },
  transform: (data) => {
    return data.filter((f) => f.hidden == 0 && f.read_only == 0)
  },
})

onMounted(() => {
  if (fields.data?.length) return
  fields.fetch()
})

const recordCount = computed(() => props.selectedValues?.size || 0)

const field = ref({
  label: '',
  fieldtype: '',
  fieldname: '',
  options: '',
})

const newValue = ref('')
const loading = ref(false)

function updateValues() {
  let fieldVal = newValue.value
  if (field.value.fieldtype == 'Check') {
    fieldVal = fieldVal == 'Yes' ? 1 : 0
  }
  loading.value = true
  call(
    'frappe.desk.doctype.bulk_update.bulk_update.submit_cancel_or_update_docs',
    {
      doctype: props.doctype,
      docnames: Array.from(props.selectedValues),
      action: 'update',
      data: {
        [field.value.fieldname]: fieldVal || null,
      },
    },
  ).then(() => {
    field.value = {
      label: '',
      fieldtype: '',
      fieldname: '',
      options: '',
    }
    newValue.value = ''
    loading.value = false
    show.value = false
    capture('bulk_update', { doctype: props.doctype })
    emit('reload')
  })
}

function changeField(f) {
  newValue.value = ''
  if (!f) return
  field.value = f
}

function updateValue(v) {
  let value = v.target ? v.target.value : v
  newValue.value = value
}

function getSelectOptions(options) {
  return options.split('\n')
}

function getValueComponent(f) {
  const { fieldtype, options } = f
  if (typeSelect.includes(fieldtype) || typeCheck.includes(fieldtype)) {
    const _options =
      fieldtype == 'Check' ? ['Yes', 'No'] : getSelectOptions(options)
    return h(FormControl, {
      type: 'select',
      options: _options.map((o) => ({
        label: o,
        value: o,
      })),
      modelValue: newValue.value,
    })
  } else if (typeLink.includes(fieldtype)) {
    if (fieldtype == 'Dynamic Link') {
      return h(FormControl, { type: 'text' })
    }
    return h(Link, { class: 'form-control', doctype: options })
  } else if (typeNumber.includes(fieldtype)) {
    return h(FormControl, { type: 'number' })
  } else if (typeDate.includes(fieldtype)) {
    return h(DatePicker)
  } else if (typeEditor.includes(fieldtype)) {
    return h(TextEditor, {
      variant: 'outline',
      editorClass:
        '!prose-sm overflow-auto min-h-[80px] max-h-80 py-1.5 px-2 rounded border border-outline-gray-2 bg-surface-white hover:border-outline-gray-3 hover:shadow-sm focus:bg-surface-white focus:border-outline-gray-4 focus:shadow-sm focus:ring-0 focus-visible:ring-2 focus-visible:ring-outline-gray-3 text-ink-gray-8 transition-colors',
      bubbleMenu: true,
      content: newValue.value,
    })
  } else {
    return h(FormControl, { type: 'text' })
  }
}
</script>
