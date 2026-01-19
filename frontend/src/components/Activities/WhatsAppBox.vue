<template>
  <div
    v-if="reply?.message"
    class="flex items-center justify-around gap-2 px-3 pt-2 sm:px-10"
  >
    <div
      class="mb-1 ml-13 flex-1 cursor-pointer rounded border-0 border-l-4 border-green-500 bg-surface-gray-2 p-2 text-base text-ink-gray-5"
      :class="reply.type == 'Incoming' ? 'border-green-500' : 'border-blue-400'"
    >
      <div
        class="mb-1 text-sm font-bold"
        :class="reply.type == 'Incoming' ? 'text-ink-green-2' : 'text-ink-blue-link'"
      >
        {{ reply.from_name || __('You') }}
      </div>
      <div class="max-h-12 overflow-hidden" v-html="reply.message" />
    </div>

    <Button variant="ghost" icon="x" @click="reply = {}" />
  </div>
  <div class="flex items-end gap-2 px-3 py-2.5 sm:px-10" v-bind="$attrs">
    <div class="flex h-8 items-center gap-2">
      <FileUploader @success="(file) => uploadFile(file)">
        <template v-slot="{ openFileSelector }">
          <div class="flex items-center space-x-2">
            <Dropdown :options="uploadOptions(openFileSelector)">
              <FeatherIcon
                name="plus"
                class="size-4.5 cursor-pointer text-ink-gray-5"
              />
            </Dropdown>
          </div>
        </template>
      </FileUploader>
      <IconPicker
        v-model="emoji"
        v-slot="{ togglePopover }"
        @update:modelValue="
          () => {
            content += emoji
            $refs.textareaRef.el.focus()
            capture('whatsapp_emoji_added')
          }
        "
      >
        <SmileIcon
          @click="togglePopover"
          class="flex size-4.5 cursor-pointer rounded-sm text-xl leading-none text-ink-gray-4"
        />
      </IconPicker>
    </div>
    <Textarea
      ref="textareaRef"
      type="textarea"
      class="min-h-8 w-full"
      :rows="rows"
      v-model="content"
      :placeholder="is24HourWindowExpired ? __('24-hour window expired. Please use template.') : placeholder"
      :disabled="is24HourWindowExpired"
      @focus="rows = 6"
      @blur="rows = 1"
      @keydown.enter.stop="(e) => sendTextMessage(e)"
    />
    <div v-if="is24HourWindowExpired" class="mt-1 text-xs text-red-600">
      {{ __('Cannot send regular messages. Last customer message was more than 24 hours ago. Please use a template.') }}
    </div>
  </div>
</template>

<script setup>
import IconPicker from '@/components/IconPicker.vue'
import SmileIcon from '@/components/Icons/SmileIcon.vue'
import { capture } from '@/telemetry'
import { createResource, Textarea, FileUploader, Dropdown } from 'frappe-ui'
import { ref, nextTick, watch, computed } from 'vue'

const props = defineProps({
  doctype: String,
})

const doc = defineModel()
const whatsapp = defineModel('whatsapp')
const reply = defineModel('reply')
const rows = ref(1)
const textareaRef = ref(null)
const emoji = ref('')

const content = ref('')
const placeholder = ref(__('Type your message here...'))
const fileType = ref('')

// Check if 24-hour window has expired
const is24HourWindowExpired = computed(() => {
  if (!whatsapp.value?.list?.data) return false
  
  // Find the last incoming message from customer
  const messages = whatsapp.value.list.data
  const incomingMessages = messages.filter(m => m.type === 'Incoming')
  
  if (incomingMessages.length === 0) return true // No customer messages means can't send
  
  // Get the most recent incoming message
  const lastIncoming = incomingMessages.reduce((latest, current) => {
    return new Date(current.creation) > new Date(latest.creation) ? current : latest
  })
  
  const lastMessageTime = new Date(lastIncoming.creation)
  const now = new Date()
  const hoursSinceLastMessage = (now - lastMessageTime) / (1000 * 60 * 60)
  
  // 23 hours 50 minutes = 23.833 hours
  return hoursSinceLastMessage > 23.833
})

function show() {
  nextTick(() => textareaRef.value.el.focus())
}

function uploadFile(file) {
  whatsapp.value.attach = file.file_url
  whatsapp.value.content_type = fileType.value
  sendWhatsAppMessage()
  capture('whatsapp_upload_file')
}

function sendTextMessage(event) {
  if (event.shiftKey) return
  if (is24HourWindowExpired.value) {
    // Prevent sending if 24-hour window expired
    return
  }
  sendWhatsAppMessage()
  textareaRef.value.el?.blur()
  content.value = ''
  capture('whatsapp_send_message')
}

async function sendWhatsAppMessage() {
  let args = {
    reference_doctype: props.doctype,
    reference_name: doc.value.name,
    message: content.value,
    to: doc.value.mobile_no,
    attach: whatsapp.value.attach || '',
    reply_to: reply.value?.name || '',
    content_type: whatsapp.value.content_type,
  }
  content.value = ''
  fileType.value = ''
  whatsapp.value.attach = ''
  whatsapp.value.content_type = 'text'
  reply.value = {}
  createResource({
    url: 'crm.api.whatsapp.create_whatsapp_message',
    params: args,
    auto: true,
  })
}

function uploadOptions(openFileSelector) {
  if (is24HourWindowExpired.value) {
    // Disable uploads if window expired
    return []
  }
  
  return [
    {
      label: __('Upload Document'),
      icon: 'file',
      onClick: () => {
        fileType.value = 'document'
        openFileSelector()
      },
    },
    {
      label: __('Upload Image'),
      icon: 'image',
      onClick: () => {
        fileType.value = 'image'
        openFileSelector('image/*')
      },
    },
    {
      label: __('Upload Video'),
      icon: 'video',
      onClick: () => {
        fileType.value = 'video'
        openFileSelector('video/*')
      },
    },
  ]
}

watch(reply, (value) => {
  if (value?.message) {
    show()
  }
})

defineExpose({ show })
</script>
