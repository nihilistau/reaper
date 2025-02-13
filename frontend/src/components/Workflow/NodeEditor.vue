<script lang="ts" setup>
import { computed, PropType, ref, watch } from 'vue'
import { XMarkIcon, FolderIcon } from '@heroicons/vue/20/solid'
import { node, workflow } from '../../../wailsjs/go/models'
import { NodeType, ParentType, NodeTypeName, ChildType } from '../../lib/Workflows'
import IDE from '../Http/IDE.vue'
import { HttpRequest } from '../../lib/Http'
import KeyValEditor from '../KeyValEditor.vue'
import { KeyValue } from '../../lib/KeyValue'
import { SelectFile } from '../../../wailsjs/go/backend/App'

const props = defineProps({
  node: { type: Object as PropType<workflow.NodeM>, required: true },
})

const safe = ref<workflow.NodeM>(safeCopy(props.node))
watch(() => props.node, n => {
  if (n) {
    safe.value = safeCopy(n)
  }
})

const emit = defineEmits(['update', 'close'])

function safeCopy(n: workflow.NodeM): workflow.NodeM {
  const c = JSON.parse(JSON.stringify(n)) as workflow.NodeM
  if (!c.name) {
    c.name = NodeTypeName(c.type as NodeType)
  }
  return c
}

function publish() {
  emit('update', safe.value)
}

const staticInputs = computed(() => safe.value?.vars?.inputs?.filter(input => {
  switch (input.type) {
    case ParentType.STRING:
      return true
    case ParentType.INT:
      return true
    case ParentType.LIST:
      return true
    case ParentType.REQUEST:
      return !input.linkable
    default:
      return !input.linkable
  }
}) || [])

function updateStringField(field: node.Connector, event: Event) {
  if (!safe.value?.vars?.static) {
    return
  }
  safe.value.vars.static[field.name].data = (event.target as HTMLInputElement).value
  publish()
}

function updateIntField(field: node.Connector, event: Event) {
  if (!safe.value?.vars?.static) {
    return
  }
  event.preventDefault()
  const el = (event.target as HTMLInputElement)
  const val = el.value
  const num = val.replace(/[^0-9]/g, '')
  safe.value.vars.static[field.name].data = parseInt(num, 10)
  el.value = num
  publish()
}

function isFieldChildType(field: node.Connector, type: ChildType) {
  const actual = safe.value.vars?.static[field.name]?.internal
  if (!actual) {
    return false
  }
  return actual === type
}

function updateListType(field: node.Connector, ev: Event) {
  if (!safe.value || !safe.value.vars?.static[field.name]) {
    return
  }
  const newType = parseInt((ev.target as HTMLSelectElement).value, 10)
  safe.value.vars.static[field.name] = new node.TransmissionM({
    type: ParentType.LIST,
    internal: newType,
    data: createDefaultListData(newType),
  })
  publish()
}

function createDefaultListData(t: ChildType) {
  switch (t) {
    case ChildType.NUMERIC_RANGE_LIST:
      return [0, 100]
    case ChildType.WORD_LIST:
      return ''
    default:
      return null
  }
}

function updateNumericRangeStart(field: node.Connector, ev: Event) {
  if (!safe.value || !safe.value.vars?.static[field.name]) {
    return
  }
  const val = parseInt((ev.target as HTMLInputElement).value, 10)
  safe.value.vars.static[field.name].data[0] = val
  publish()
}

function updateNumericRangeEnd(field: node.Connector, ev: Event) {
  if (!safe.value || !safe.value.vars?.static[field.name]) {
    return
  }
  const val = parseInt((ev.target as HTMLInputElement).value, 10)
  safe.value.vars.static[field.name].data[1] = val
  publish()
}

const requestActions = new Map<string, string>([])

function updateRequestField(field: node.Connector, req: HttpRequest) {
  if (!safe.value || !safe.value.vars?.static[field.name]) {
    return
  }
  safe.value.vars.static[field.name].data = req
  publish()
}

function updateMapField(field: node.Connector, kvs: KeyValue[]) {
  if (!safe.value || !safe.value.vars?.static[field.name]) {
    return
  }
  const m = new Map<string, string>([])
  kvs.forEach(kv => {
    m.set(kv.Key, kv.Value)
  })
  safe.value.vars.static[field.name].data = Object.fromEntries(m)
  publish()
}

function keyValsFromMap(field: node.Connector): KeyValue[] {
  const data = safe.value.vars?.static[field.name]?.data
  if (data) {
    return Object.entries(data).map(([k, v]) => ({
      Key: k,
      Value: v,
    } as KeyValue))
  }
  return []
}

function updateBooleanField(field: node.Connector, ev: Event) {
  if (!safe.value || !safe.value.vars?.static[field.name]) {
    return
  }
  const val = (ev.target as HTMLInputElement).checked
  safe.value.vars.static[field.name].data = val
  publish()
}

function getLabel(field: node.Connector) {
  const label = field.name.replace(/_/g, ' ')
  if (!field.description) {
    return label
  }
  return `${label} (${field.description})`
}

function getBase(path: string): string {
  const parts = path.split('/')
  return parts.pop() as string
}

function updateWordList(field: node.Connector) {
  if (!safe.value || !safe.value.vars) {
    return
  }
  SelectFile('Select wordlist').then(
    (path: string) => {
      if (!path) {
        return
      }
      if (!safe.value.vars) {
        safe.value.vars = new node.VarStorageM({})
      }
      safe.value.vars.static[field.name].data = path
      publish()
    },
    (err: Error) => {
      throw err
    },
  )
}

</script>

<template>
  <div
      class="border rounded border-polar-night-3 relative p-2 bg-polar-night-1 text-center max-h-full overflow-y-auto pointer-events-auto">
    <button @click="emit('close')" class="absolute right-1 top-1">
      <XMarkIcon class="w-4 h-4"/>
    </button>
    {{ NodeTypeName(safe.type) }}
    <div class="relative mt-2 text-left">
      <!-- FORM BEGIN -->

      <!-- GLOBAL OPTIONS -->
      <div class="mt-2">
        <div class="sm:col-span-4">
          <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">Name</label>
          <div class="mt-1">
            <div
                class="flex rounded-md bg-white/5 ring-1 ring-inset ring-white/10 focus-within:ring-2 focus-within:ring-inset focus-within:ring-indigo-500">
              <input type="text" autocomplete="off" autocapitalize="off" spellcheck="false"
                     v-model="safe.name"
                     @input="publish"
                     class="flex-1 border-0 bg-transparent py-1.5 px-2 text-snow-storm-1 focus:ring-0 sm:text-sm sm:leading-6"/>
            </div>
          </div>
        </div>
      </div>

      <div v-for="field in staticInputs" class="mt-2" :key="field.name">
        <div v-if="field.type === ParentType.STRING" class="sm:col-span-4">
          <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">{{
              getLabel(field)
            }}</label>
          <div class="mt-1">
            <div
                class="flex rounded-md bg-white/5 ring-1 ring-inset ring-white/10 focus-within:ring-2 focus-within:ring-inset focus-within:ring-indigo-500">
              <input type="text" autocomplete="off" autocapitalize="off" spellcheck="false"
                     :value="safe.vars?.static[field.name].data"
                     @input="updateStringField(field, $event)"
                     class="flex-1 border-0 bg-transparent py-1.5 px-2 text-snow-storm-1 focus:ring-0 sm:text-sm sm:leading-6"/>
            </div>
          </div>
        </div>
        <div v-else-if="field.type === ParentType.INT" class="sm:col-span-4">
          <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">{{
              getLabel(field)
            }}</label>
          <div class="mt-1">
            <div
                class="flex rounded-md bg-white/5 ring-1 ring-inset ring-white/10 focus-within:ring-2 focus-within:ring-inset focus-within:ring-indigo-500">
              <input type="number" autocomplete="off" autocapitalize="off"
                     spellcheck="false"
                     :value="safe.vars?.static[field.name].data"
                     @input="updateIntField(field, $event)"
                     class="flex-1 border-0 bg-transparent py-1.5 px-2 text-snow-storm-1 focus:ring-0 sm:text-sm sm:leading-6"/>
            </div>
          </div>
        </div>
        <div v-else-if="field.type === ParentType.LIST" class="sm:col-span-4">
          <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">{{
              getLabel(field)
            }}</label>
          <div class="mt-1">
            <div
                class="flex rounded-md bg-white/5 ring-1 ring-inset ring-white/10 focus-within:ring-2 focus-within:ring-inset focus-within:ring-indigo-500">
              <select
                  @change="updateListType(field, $event)"
                  class="flex-1 border-0 bg-transparent py-1.5 px-2 text-snow-storm-1 focus:ring-0 sm:text-sm sm:leading-6">
                <option :selected="isFieldChildType(field, ChildType.NUMERIC_RANGE_LIST)"
                        :value="ChildType.NUMERIC_RANGE_LIST">
                  Numeric Range
                </option>
                <option :selected="isFieldChildType(field, ChildType.WORD_LIST)"
                        :value="ChildType.WORD_LIST">
                  Wordlist
                </option>
                <option :selected="isFieldChildType(field, ChildType.COMMA_SEP_LIST)"
                        :value="ChildType.COMMA_SEP_LIST">
                  Comma Separated Values
                </option>
              </select>
            </div>
          </div>
          <div class="mt-1">
            <div v-if="isFieldChildType(field, ChildType.NUMERIC_RANGE_LIST)">
              <div class="mt-1">
                <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">Start</label>
                <div class="mt-1">
                  <div
                      class="flex rounded-md bg-white/5 ring-1 ring-inset ring-white/10 focus-within:ring-2 focus-within:ring-inset focus-within:ring-indigo-500">
                    <input type="number" autocomplete="off" autocapitalize="off"
                           spellcheck="false"
                           :value="safe.vars?.static[field.name].data[0]"
                           @input="updateNumericRangeStart(field, $event)"
                           class="flex-1 border-0 bg-transparent py-1.5 px-2 text-snow-storm-1 focus:ring-0 sm:text-sm sm:leading-6"/>
                  </div>
                </div>
              </div>
              <div class="mt-1">
                <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">End (inclusive)</label>
                <div class="mt-1">
                  <div
                      class="flex rounded-md bg-white/5 ring-1 ring-inset ring-white/10 focus-within:ring-2 focus-within:ring-inset focus-within:ring-indigo-500">
                    <input type="number" autocomplete="off" autocapitalize="off"
                           spellcheck="false"
                           :value="safe.vars?.static[field.name].data[1]"
                           @input="updateNumericRangeEnd(field, $event)"
                           class="flex-1 border-0 bg-transparent py-1.5 px-2 text-snow-storm-1 focus:ring-0 sm:text-sm sm:leading-6"/>
                  </div>
                </div>
              </div>
            </div>
            <div v-else-if="isFieldChildType(field, ChildType.WORD_LIST)"
                 class="text-snow-storm-1/80 mt-2">
              <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">Wordlist</label>
              <div class="flex py-1 text-sm border rounded border-polar-night-4 bg-white/5 mt-1">
                <button @click="updateWordList(field)" class="mx-2 flex-shrink">
                  <FolderIcon class="h-4 w-4 text-snow-storm-1"/>
                </button>
                <div class="flex-grow pt-1 cursor-pointer" @click="updateWordList(field)">
                  <p
                      v-if="safe.vars?.static[field.name].data">{{
                      getBase(safe.vars?.static[field.name].data)
                    }}</p>
                  <p class="italic" v-else>No file selected</p>
                </div>
              </div>

            </div>
            <div v-else-if="isFieldChildType(field, ChildType.COMMA_SEP_LIST)"
                 class="text-snow-storm-1/80 mt-2">
              <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">Comma Separated
                Values</label>
              <div class="mt-1">
                <div
                    class="flex rounded-md bg-white/5 ring-1 ring-inset ring-white/10 focus-within:ring-2 focus-within:ring-inset focus-within:ring-indigo-500">
                  <input type="text" autocomplete="off" autocapitalize="off" spellcheck="false"
                         :value="safe.vars?.static[field.name].data"
                         @input="updateStringField(field, $event)"
                         class="flex-1 border-0 bg-transparent py-1.5 px-2 text-snow-storm-1 focus:ring-0 sm:text-sm sm:leading-6"/>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div v-else-if="field.type === ParentType.BOOLEAN" class="sm:col-span-4">
          <div class="relative flex items-start">
            <div class="flex h-6 items-center">
              <input :id="field.name" :name="field.name" type="checkbox"
                     :checked="safe.vars?.static[field.name].data"
                     @change="updateBooleanField(field, $event)"
                     class="h-4 w-4 ml-2 bg-polar-night-4 rounded text-frost-1 focus:text-frost-1"/>
            </div>
            <div class="ml-2 text-sm leading-6">
              <label :for="field.name" class="font-medium text-snow-storm-1 capitalize">{{
                  getLabel(field)
                }}</label>
            </div>
          </div>
        </div>
        <div v-else-if="field.type === ParentType.REQUEST" class="sm:col-span-4">
          <IDE :request="safe.vars?.static[field.name].data" :actions="requestActions"
               :readonly="false" :show-buttons="false"
               @request-update="updateRequestField(field, $event)"/>
        </div>
        <div v-else-if="field.type === ParentType.MAP" class="sm:col-span-4">
          <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">{{
              getLabel(field)
            }}</label>
          <KeyValEditor :data="keyValsFromMap(field)"
                        :readonly="false"
                        @publish="updateMapField(field, $event)"/>
        </div>
        <div v-else class="sm:col-span-4">
          <label class="block text-sm font-medium leading-6 text-snow-storm-1 capitalize">{{ field.name }}</label>
          <div class="mt-1">
            <i>This value cannot be edited. This is a bug!</i>
          </div>
        </div>
      </div>
      <!-- FORM END -->
    </div>
  </div>
</template>
