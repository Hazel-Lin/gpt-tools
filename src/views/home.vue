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
  },
  {
    icon: '😁',
    name: '小工具',
    p: '将这个对象生成一个对象数组，其中的key翻译成中文作为label的值，其中的value作为‘value’的值放入到这个对象数组中，请直接返回代码，无需太多过程描述。',
    active: false,
  },
])

async function sendChatMessage(content: string = messageContent.value) {
  try {
    isTalking.value = true
    if (messageList.value.length === 2)
      messageList.value.pop()
    messageList.value.push({ role: 'user', content: prompt.value })
    messageList.value.push({ role: 'user', content })
    clearMessageContent()
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
function handleClick(item) {
  prompt.value = item.p
  toolList.value.map(tool => tool.active = false)
  item.active = true
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
          :type="isConfig ? 'password' : 'text'"
          placeholder="Please input your key:sk-xxxxxxxxxx"
          @keydown.enter="sendOrSave()"
        >
        <button text-white :disabled="!APIKeyContent" bg-pink rounded-md p-2 box-border @click="sendOrSave()">
          save
        </button>
      </div>
    </div>
    <div flex mb10px class="new-chat_mask">
      <div
        v-for="item, index in toolList"
        :key="index"
        flex
        items-center
        class="new-chat_mask__HbHeX"
        bg="#1e1e1e"
        mb10
        mr10px
        px14px
        py10px
        :class="{ 'new-chat_mask--active': item.active }"
        @click="handleClick(item)"
      >
        <div class="user-avatar" h30px w30px flex items-center justify-center>
          {{ item.icon }}
        </div>
        <div ml10px class="one-line" color="#bbb">
          {{ item.name }}
        </div>
      </div>
    </div>
    <div flex justify-between mt-3>
      <div w="45%">
        <textarea v-model="messageContent" w="100%" w-full h-500px bg="#1e1e1e" shadow-md rounded-md outline-none p-2 box-border text-white placeholder="Type something..." />
      </div>
      <div flex items-center>
        <button text-white :disabled="isTalking" bg-pink rounded-md p-2 box-border @click="sendChatMessage">
          Answer
        </button>
      </div>
      <div w="45%">
        <div ref="chatListDom" w-full h-200px bg="#1e1e1e" text-white shadow-md rounded-md p-2 h-500px>
          <div
            v-for="item, index of messageList.filter((v) => v.role === 'assistant')"
            :key="index"
          >
            <div
              v-if="item.content"
              class="text-sm leading-relaxed text-slate-600"
              v-html="md.render(item.content)"
            />
            <div
              v-else
              class="text-sm leading-relaxed text-slate-600"
            >
              Hello, I'm ChatGPT
            </div>
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
    border-radius: 10px;
    max-width: 8em;
    transform: scale(1);
    cursor: pointer;
    transition: all .3s ease;

}
 .new-chat_mask :hover {
    transform: translateY(-5px) scale(1.1);
    z-index: 999;
    border-color: pink;
}
.user-avatar{
    min-height: 30px;
    min-width: 30px;
    border: 1px solid hsla(0,0%,100%,.192);
    box-shadow: 0px 2px 4px 0px rgba(0,0,0,.05);
    border-radius: 10px;
}
.one-line{
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.new-chat_mask--active{
  border: 1px solid pink;
}
</style>
