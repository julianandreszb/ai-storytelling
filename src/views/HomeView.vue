<script setup lang="ts">
import { nextTick, reactive, ref } from 'vue'
//TODO: Resize image
import fairytaleIcon from '@/assets/images/fairytale.png'
import aiIcon from '@/assets/images/ai.png'

const workerAiHost = import.meta.env.VITE_WORKER_AI_HOST
const isChatSectionVisible = ref(false)
const isChatFormEnabled = ref(true)
const isEndStoryButtonEnabled = ref(false)
const chatStory: string[] = []
const useFullStoryForImageGenerator = ref(false)

//TODO: Move to constants
const aiStoryPrompts = [
  {
    role: 'system',
    content: 'You are a story telling assistant'
  },
  {
    role: 'user',
    content:
      'Create the first part of a tale and wait for my story contribution part, we will continue in this way until we complete the story, keep each story part concise and under 200 characters, do not include the prompt text on the response, do not repeat the first part of the story on new requests. Start the story'
  }
]

//TODO: Move to constants
const timeoutMessage = 2000

//TODO: Move to interfaces
interface Image {
  path: string
  alt: string
}

//TODO: Move to interfaces
interface Message {
  id: number
  sender: 'user' | 'ai'
  content: string
  iconFront?: Image | null
  iconBack?: Image | null
  image?: HTMLImageElement | null
}

const userInput = ref<string>('')
const messages = reactive<Message[]>([])

//TODO: Move to utils
function scrollToBottom() {
  window.scrollTo(0, document.body.scrollHeight)
}

//TODO: Move to utils
function timeout(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

const frontIcon: Image = { path: fairytaleIcon, alt: 'Fairytale Image' }
const backIcon: Image = { path: aiIcon, alt: 'Fairytale Image' }

function getStoryAsText(chatStory: string[]) {
  return chatStory.join(' ')
}

async function explainGame() {
  await timeout(timeoutMessage)
  addMessage(
    'ai',
    `Let's create a story together. I'll begin with the first part`,
    frontIcon,
    backIcon
  )
  await timeout(timeoutMessage)
  addMessage('ai', `Then you can carry it forward`, null, backIcon)
}
function disableChatForm() {
  isEndStoryButtonEnabled.value = false
  isChatFormEnabled.value = false
}

function enableChatForm() {
  isEndStoryButtonEnabled.value = true
  isChatFormEnabled.value = true
}

async function startStoryTelling() {
  //1. Greet the user
  addMessage('ai', `Hi there!`, null, backIcon)

  //2. Explain the game
  await explainGame()

  //3. Start the story and wait for user input
  const responseFirstPart = await getStoryPart()
  const imagePromptText = useFullStoryForImageGenerator.value
    ? getStoryAsText(chatStory)
    : responseFirstPart
  const img = await createImage(imagePromptText)
  addMessage('ai', responseFirstPart, null, backIcon, img)
  isChatSectionVisible.value = true
  isEndStoryButtonEnabled.value = true
}

async function createImage(prompt: string): Promise<HTMLImageElement | null> {
  //TODO: Move to useApi (Composable)
  try {
    const response = await fetch(`${workerAiHost}/image?query=${prompt}`)
    const blob = await response.blob()
    console.log(blob)
    const img = document.createElement('img')
    img.src = URL.createObjectURL(blob)

    return img
  } catch (error) {
    console.error('Error:', error)
    return null
  }
}

async function getStoryPart() {
  //TODO: Move to useApi (Composable)
  const response = await fetch(`${workerAiHost}/entry`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(aiStoryPrompts)
  })

  const data = await response.json()
  chatStory.push(data.response)
  return data.response
}

function addMessage(
  sender: 'user' | 'ai',
  content: string,
  iconFront?: Image | null,
  iconBack?: Image | null,
  image?: HTMLImageElement | null
) {
  messages.push({ id: messages.length + 1, sender, content, iconFront, iconBack, image })
  nextTick(() => {
    scrollToBottom()
  })
}

async function handleSubmit() {
  disableChatForm()

  const prompt = userInput.value
  userInput.value = ''

  chatStory.push(prompt)
  let imagePromptText = useFullStoryForImageGenerator.value ? getStoryAsText(chatStory) : prompt
  const imgUserInput = await createImage(imagePromptText)
  addMessage('user', prompt, null, null, imgUserInput)

  aiStoryPrompts.push({
    role: 'user',
    content: `${prompt}. Continue with the next part`
  })
  const responseFirstPart = await getStoryPart()
  imagePromptText = useFullStoryForImageGenerator.value
    ? getStoryAsText(chatStory)
    : responseFirstPart
  const img = await createImage(imagePromptText)
  addMessage('ai', responseFirstPart, null, null, img)

  enableChatForm()
}

async function endStory() {
  disableChatForm()
  isChatSectionVisible.value = false

  // const prompt = messages[messages.length - 1].content // Last prompt
  //const prompt = getStoryAsText(chatStory) // Full prompt

  aiStoryPrompts.push({
    role: 'user',
    content: `Create a happy short story final for this story`
  })
  const responseStoryEnding = await getStoryPart()
  const img = await createImage(responseStoryEnding)
  addMessage('user', responseStoryEnding, null, null, img)

  //TODO: Disable chat form and show create new history button
}

startStoryTelling()
</script>

<template>
  <main>
    <section class="chat">
      <div
        class="chat-bubble"
        :class="{ ai: message.sender === 'ai' }"
        v-for="message in messages"
        :key="message.id"
      >
        <img
          class="message-image"
          v-if="message.image"
          :src="message.image.src"
          :alt="message.image.alt"
        />
        <div>
          <img
            class="message-icon-pre"
            v-if="message.iconBack"
            :src="message.iconBack.path"
            :alt="message.iconBack.alt"
          />
          {{ message.content }}
          <img
            class="message-icon"
            v-if="message.iconFront"
            :src="message.iconFront.path"
            :alt="message.iconFront.alt"
          />
        </div>
      </div>
    </section>
    <section v-if="isChatSectionVisible" class="chat-entry">
      <form class="chat-input" @submit.prevent="handleSubmit">
        <label for="message">Write the next story part:</label>
        <textarea
          :disabled="!isChatFormEnabled"
          v-model="userInput"
          id="message"
          name="message"
          rows="3"
        ></textarea>
        <button :disabled="!isChatFormEnabled || !userInput.length" class="btn" type="submit">
          Send
        </button>
        <button
          :disabled="!isChatFormEnabled || !isEndStoryButtonEnabled"
          class="btn"
          type="button"
          @click="endStory"
        >
          End Story
        </button>
      </form>
    </section>
  </main>
</template>

<style scoped>
@keyframes slideIn {
  0% {
    opacity: 0;
    transform: translateY(20px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

main {
  display: flex;
  flex-direction: column;
  gap: 40px;
  margin: 20px;
}

.chat-bubble {
  align-content: start;
  align-items: center;
  animation: slideIn 0.5s ease forwards;
  background-color: #007bff; /* Adjust colors as needed */
  border-radius: 20px;
  color: #fff;
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  margin-bottom: 20px;
  padding: 10px 15px;
  position: relative;
  width: 600px;
}

.chat-bubble.ai {
  align-self: flex-start;
  background-color: #f0f0f0;
  color: #000;
}

.message-icon {
  height: 20px;
  margin-inline-start: 0.5rem;
}

.message-icon-pre {
  height: 20px;
  margin-inline-start: 0.5rem;
}

.message-image {
  max-width: 100%;
  height: auto;
  border-radius: 10px;
  margin: 10px 0;
}

.chat-input {
  width: 600px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.chat-input .btn {
  background-color: #007bff;
  border: none;
  border-radius: 5px;
  color: white;
  cursor: pointer;
  padding: 10px 20px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 16px;
  margin: 4px 2px;
  transition-duration: 0.4s;
}

.chat-input .btn:hover {
  background-color: #0066cc;
  color: white;
}

.chat-input .btn:disabled {
  background-color: #cccccc;
  color: #666666;
  cursor: not-allowed;
}
</style>
