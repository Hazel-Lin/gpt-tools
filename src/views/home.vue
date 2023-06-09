<script setup lang="ts">
import cryptoJS from 'crypto-js'
import type { ChatMessage } from '@/types'
import { chat, md } from '@/libs'

let apiKey = ''
const isConfig = ref(true)
const isTalking = ref(false)
const messageContent = ref('')
const chatListDom = ref<HTMLDivElement>()
const decoder = new TextDecoder('utf-8')
const messageList = ref<ChatMessage[]>([
  {
    role: 'assistant',
    content: '',
  },
])

onMounted(() => {
  if (getAPIKey())
    switchConfigStatus()
})
const toolList = ref([
  {
    icon: '🍠',
    name: '小红书',
    p: '用小红书的风格针对这段内容做修改，要求内容生动。',
    active: false,
    placeholder: '请输入一段文案，我们将用小红书的风格修饰它',
  },
  {
    icon: '😁',
    name: '小工具',
    p: '将这个对象生成一个对象数组，其中的key翻译成中文作为label的值，其中的value作为‘value’的值放入到这个对象数组中，请直接返回代码，无需太多过程描述。',
    active: false,
    placeholder: '请直接输入一个对象',
  },
])
const isShowHint = ref(true)

async function sendChatMessage(content: string = messageContent.value) {
  try {
    isTalking.value = true
    isShowHint.value = false
    if (messageList.value.length === 2)
      messageList.value.pop()
    messageList.value.push({ role: 'user', content: prompt.value })
    messageList.value.push({ role: 'user', content })
    // clearMessageContent()
    messageList.value.push({ role: 'assistant', content: '' })

    const { body, status } = await chat(messageList.value, getAPIKey())
    if (body) {
      const reader = body.getReader()
      await readStream(reader, status)
    }
  }
  catch (error: any) {
    appendLastMessageContent(error)
  }
  finally {
    isTalking.value = false
  }
}

async function readStream(reader: ReadableStreamDefaultReader<Uint8Array>,
  status: number) {
  let partialLine = ''

  while (true) {
    const { value, done } = await reader.read()
    if (done)
      break

    const decodedText = decoder.decode(value, { stream: true })

    if (status !== 200) {
      const json = JSON.parse(decodedText) // start with "data: "
      const content = json.error.message ?? decodedText
      appendLastMessageContent(content)
      return
    }

    const chunk = partialLine + decodedText
    const newLines = chunk.split(/\r?\n/)

    partialLine = newLines.pop() ?? ''

    for (const line of newLines) {
      if (line.length === 0)
        continue // ignore empty message
      if (line.startsWith(':'))
        continue // ignore sse comment message
      if (line === 'data: [DONE]')
        return //

      const json = JSON.parse(line.substring(6)) // start with "data: "
      const content
        = status === 200
          ? json.choices[0].delta.content ?? ''
          : json.error.message
      appendLastMessageContent(content)
    }
  }
}
// 重置key
function resetKey() {
  localStorage.setItem('apiKey', '')
  apiKey = ''
  isConfig.value = true
}

function appendLastMessageContent(content: string) {
  return messageList.value[messageList.value.length - 1].content += content
}
const APIKeyContent = ref('')
function sendOrSave() {
  if (!APIKeyContent.value.length)
    return
  if (isConfig.value) {
    if (saveAPIKey(APIKeyContent.value.trim()))
      switchConfigStatus()
    // 清空api key
    clearKeyContent()
  }
}

const getSecretKey = () => 'hazel'

function saveAPIKey(apiKey: string) {
  if (apiKey.slice(0, 3) !== 'sk-' || apiKey.length !== 51) {
    alert('API Key 错误，请检查后重新输入！')
    return false
  }
  const aesAPIKey = cryptoJS.AES.encrypt(apiKey, getSecretKey()).toString()
  localStorage.setItem('apiKey', aesAPIKey)
  return true
}

function getAPIKey() {
  if (apiKey)
    return apiKey
  const aesAPIKey = localStorage.getItem('apiKey') ?? ''
  apiKey = cryptoJS.AES.decrypt(aesAPIKey, getSecretKey()).toString(
    cryptoJS.enc.Utf8,
  )
  return apiKey
}

const switchConfigStatus = () => (isConfig.value = !isConfig.value)

const clearMessageContent = () => (messageContent.value = '')
const clearKeyContent = () => (APIKeyContent.value = '')

function scrollToBottom() {
  if (!chatListDom.value)
    return
  scrollTo(0, chatListDom.value.scrollHeight)
}

watch(messageList.value, () => nextTick(() => scrollToBottom()))

const prompt = ref('')
const placeholder = ref('')
function handleClick(item) {
  if (messageContent.value)
    messageContent.value = ''
  prompt.value = item.p
  placeholder.value = item.placeholder
  toolList.value.map(tool => tool.active = false)
  item.active = true
}
function handleCancel() {
  prompt.value = ''
  placeholder.value = ''
  toolList.value.map(tool => tool.active = false)
}
</script>

<template>
  <div bg="#151515" h-screen py-10 px-30>
    <div class="sticky bottom-0 w-full pb-8">
      <div v-if="isConfig" text-white mb-2>
        Please input your key：
      </div>
      <div class="flex" w="52%">
        <input
          v-model="APIKeyContent"
          class="input"
          type="password"
          placeholder="Please input your key:sk-xxxxxxxxxx"
          @keydown.enter="sendOrSave()"
        >
        <button text-white bg-pink rounded-md p-2 box-border @click="apiKey ? resetKey() : sendOrSave()">
          {{ apiKey ? 'reset' : 'save' }}
        </button>
      </div>
    </div>
    <div flex>
      <div flex mb10px class="new-chat_mask">
        <div
          v-for="item, index in toolList"
          :key="index"
          flex
          items-center
          max-w-8em
          rd-10px
          class="new-chat_mask__HbHeX"
          cursor-pointer
          bg="#1e1e1e"
          mb10
          mr10px
          px14px
          py10px
          :class="{ 'new-chat_mask--active': item.active }"
          @click="handleClick(item)"
        >
          <div class="user-avatar" min-h-30px min-w-30px rd-10px h30px w30px flex items-center justify-center>
            {{ item.icon }}
          </div>
          <div ml10px class="one-line" color="#bbb">
            {{ item.name }}
          </div>
        </div>
      </div>
      <button
        text-white
        mb10
        mr10px
        px14px
        py10px
        h50px
        bg-pink
        rounded-md
        box-border
        :disabled="!prompt"
        @click="handleCancel"
      >
        Cancel
      </button>
    </div>
    <div flex justify-between mt-3>
      <div w="45%">
        <textarea v-model="messageContent" text-slate-600 w="100%" fs-14 w-full h-500px bg="#1e1e1e" text-size-14px shadow-md rounded-md outline-none p-2 box-border :placeholder="placeholder || 'Please input your message'" />
      </div>
      <div flex items-center>
        <button text-white :disabled="isTalking || !apiKey" bg-pink rounded-md p-2 box-border @click="sendChatMessage()">
          Answer
        </button>
      </div>
      <div w="45%">
        <div ref="chatListDom" w-full h-200px bg="#1e1e1e" text-white shadow-md rounded-md p-2 h-500px overflow-y-auto>
          <div
            v-if="isShowHint"
            class="text-sm leading-relaxed text-slate-600"
          >
            Hello, I'm ChatGPT-3.5, how can I help you?
          </div>
          <div
            v-for="item, index of messageList.filter((v) => v.role === 'assistant')"
            :key="index"
          >
            <div
              v-if="item.content"
              class="text-sm leading-relaxed text-slate-600"
              v-html="md.render(item.content)"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
pre {
  font-family: -apple-system, "Noto Sans", "Helvetica Neue", Helvetica,
    "Nimbus Sans L", Arial, "Liberation Sans", "PingFang SC", "Hiragino Sans GB",
    "Noto Sans CJK SC", "Source Han Sans SC", "Source Han Sans CN",
    "Microsoft YaHei", "Wenquanyi Micro Hei", "WenQuanYi Zen Hei", "ST Heiti",
    SimHei, "WenQuanYi Zen Hei Sharp", sans-serif;
}

.new-chat_mask__HbHeX{
    border: 1px solid hsla(0,0%,100%,.192);
    box-shadow: 0px 2px 4px 0px rgba(0,0,0,.05);
    transform: scale(1);
    transition: all .3s ease;

}
 .new-chat_mask :hover {
    transform: translateY(-5px) scale(1.1);
    z-index: 999;
    border-color: pink;
}
.user-avatar{
  border: 1px solid hsla(0,0%,100%,.192);
  box-shadow: 0px 2px 4px 0px rgba(0,0,0,.05);
}
.one-line{
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.new-chat_mask--active{
  border: 1px solid pink;
}
textarea{
  resize: none;
}
</style>
