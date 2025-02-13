<script lang="ts" setup>
import { PropType } from 'vue'
import { TransitionChild, TransitionRoot, Dialog, DialogPanel } from '@headlessui/vue'
import SettingsComponent from './SettingsEditor.vue'
import Settings from '../lib/Settings'
import { backend } from '../../wailsjs/go/models'
import VersionInfo = backend.VersionInfo;

const props = defineProps({
  show: {
    type: Boolean,
    required: true,
  },
  settings: { type: Object as PropType<Settings>, required: true },
  version: { type: Object as PropType<VersionInfo | null>, required: true },
})

const emit = defineEmits(['save', 'close'])

function close() {
  emit('close')
}

function saveSettings(settings: Settings) {
  emit('save', settings)
}
</script>

<template>
  <TransitionRoot as="template" :show="props.show">
    <Dialog as="div" class="relative z-10" @close="close">
      <TransitionChild
          as="template"
          enter="ease-out duration-300"
          enter-from="opacity-0"
          enter-to="opacity-100"
          leave="ease-in duration-200"
          leave-from="opacity-100"
          leave-to="opacity-0">
        <div class="fixed inset-0 bg-gray-700 bg-opacity-75 transition-opacity"/>
      </TransitionChild>

      <div class="fixed inset-0 z-10 overflow-y-auto">
        <div class="mt-10 min-h-full items-end justify-center p-4 text-center sm:items-center sm:p-0">
          <TransitionChild
              as="template"
              enter="ease-out duration-300"
              enter-from="opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95"
              enter-to="opacity-100 translate-y-0 sm:scale-100"
              leave="ease-in duration-200"
              leave-from="opacity-100 translate-y-0 sm:scale-100"
              leave-to="opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95">
            <DialogPanel class="relative transform overflow-hidden text-left transition-all">
              <SettingsComponent @save="saveSettings" @cancel="close" :settings="props.settings"
                                 :version="props.version"/>
            </DialogPanel>
          </TransitionChild>
        </div>
      </div>
    </Dialog>
  </TransitionRoot>
</template>
