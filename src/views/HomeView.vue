<script setup lang="ts">
import { nextTick, reactive, ref } from 'vue'

//TODO: Resize-optimize images
import aiIcon from '@/assets/images/ai-2.png'
import spanishIcon from '@/assets/images/es.png'
import englishIcon from '@/assets/images/en.png'

const workerAiHost = import.meta.env.VITE_WORKER_AI_HOST
const isChatSectionVisible = ref(false)
const isChatFormEnabled = ref(true)
const isEndStoryButtonEnabled = ref(false)
const preferredLanguage = ref('english')
const isLanguageSectionVisible = ref(true)

const isLoadingStory = ref(false)
const isGeneratingStoryPart = ref(false)
const isGeneratingImage = ref(false)

//TODO: Move to constants
const storyTypes = [
  'Fairy Tales',
  'Fables',
  'Adventure',
  'Science Fiction',
  'Mystery',
  'Fantasy',
  'Mythology',
  'Superhero',
  'Animal',
  'Ghost',
  'Comedy',
  'Drama',
  'Action and Adventure',
  'Magic and Supernatural',
  'Horror',
  'Romance'
]

const randomIndex = Math.floor(Math.random() * storyTypes.length)
const randomStoryType = storyTypes[randomIndex]

//TODO: Move to constants
const aiStoryPrompts = [
  // {
  //   role: 'system',
  //   content: 'You are a story telling assistant'
  // },
  // {
  //   role: 'user',
  //   // content: `Create the first part about ${randomStoryType} and wait for my story contribution part, we will continue in this way until we complete the story, keep each story part concise and under 200 characters, do not include the prompt request text on the response, do not repeat the first part of the story on new requests. Start the story`
  //
  //   // content: `Create the first part of a random story about ${randomStoryType} and wait for my story contribution part, we will continue in this way until we complete the story, keep each story under 200 characters. Start the story`
  //
  //   content: `Create the first part about ${randomStoryType} and wait for my story contribution part and we will continue in this way until we complete the story and keep each story part concise and under 200 characters and do not include the request on the text response. Start the story`
  // },
  // @cf/meta/llama-2-7b-chat-fp16
  {
    role: 'system',
    content: 'You are a story telling assistant'
  },
  {
    role: 'user',
    content: `Create the first part of a story about ${randomStoryType}`
  },
  {
    role: 'user',
    content: `Wait for my story contribution part and we will continue in this way until we complete the story`
  },
  {
    role: 'user',
    content: `Keep each story part concise and under 200 characters`
  },
  {
    role: 'user',
    content: `Do not include the request on the text response`
  },
  {
    role: 'user',
    content: `Start the story`
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
const visibleMessages = reactive<Message[]>([])

//TODO: Move to utils
function scrollToBottom() {
  window.scrollTo(0, document.body.scrollHeight)
}

//TODO: Move to utils
function timeout(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

const backIcon: Image = { path: aiIcon, alt: 'AI Icon' }

async function startIntro() {
  // await addMessage('ai', `Hi!`, null, backIcon)

  // await timeout(timeoutMessage)
  await addMessage(
    'ai',
    `Hi! Let's create a story together. I'll begin with the first part.`,
    null,
    backIcon
  )

  await timeout(timeoutMessage)
  await addMessage(
    'ai',
    `Then you continue, and so on until the story is finished.`,
    null,
    backIcon
  )
}

function disableChatForm() {
  isEndStoryButtonEnabled.value = false
  isChatFormEnabled.value = false
}

function enableChatForm() {
  isEndStoryButtonEnabled.value = true
  isChatFormEnabled.value = true
}

async function translate(message: string, sourceLang: string, targetLang: string): Promise<string> {
  try {
    const response = await fetch(
      `${workerAiHost}/translate?source_lang=${sourceLang}&target_lang=${targetLang}&text=${message}`,
      {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json'
        }
      }
    )

    const data = await response.json()
    return data.translated_text
  } catch (error) {
    console.error('Error:', error)
    return `Error translating message: ${error}`
  }
}

async function startStoryTelling() {
  await startIntro()

  await timeout(1000)
  isGeneratingStoryPart.value = true
  isLoadingStory.value = true
  const responseFirstPart = await getStoryPart()
  isGeneratingStoryPart.value = false
  isGeneratingImage.value = true
  const img = await createImage(responseFirstPart)
  isLoadingStory.value = false
  await addMessage('ai', responseFirstPart, null, backIcon, img)

  isChatSectionVisible.value = true
  isEndStoryButtonEnabled.value = true
}

async function createImage(prompt: string): Promise<HTMLImageElement | null> {
  //TODO: Move to useApi (Composable)
  try {
    const response = await fetch(`${workerAiHost}/image?query=${prompt}`)
    const blob = await response.blob()
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
  return data.response
}

async function addMessage(
  sender: 'user' | 'ai',
  content: string,
  iconFront?: Image | null,
  iconBack?: Image | null,
  image?: HTMLImageElement | null
) {
  if (sender === 'ai' && preferredLanguage.value !== 'english') {
    content = await translate(content, 'english', preferredLanguage.value)
  }

  visibleMessages.push({
    id: visibleMessages.length + 1,
    sender,
    content,
    iconFront,
    iconBack,
    image
  })

  await nextTick(() => {
    scrollToBottom()
  })
}

async function handleSubmit() {
  disableChatForm()

  let originalPrompt = userInput.value
  let translatedPrompt = originalPrompt
  userInput.value = ''

  isGeneratingStoryPart.value = true
  isLoadingStory.value = true

  if (preferredLanguage.value !== 'english') {
    translatedPrompt = await translate(originalPrompt, preferredLanguage.value, 'english')
  }

  isGeneratingStoryPart.value = false
  isGeneratingImage.value = true

  const imgUserInput = await createImage(translatedPrompt)
  await addMessage('user', originalPrompt, null, null, imgUserInput)

  await timeout(timeoutMessage)
  isLoadingStory.value = false

  aiStoryPrompts.push({
    role: 'user',
    content: `${translatedPrompt}. Continue with the next part.`
  })
  const responseFirstPart = await getStoryPart()
  const img = await createImage(responseFirstPart)
  await addMessage('ai', responseFirstPart, null, null, img)

  enableChatForm()
}

async function endStory() {
  disableChatForm()
  isChatSectionVisible.value = false

  aiStoryPrompts.push({
    role: 'user',
    content: `Create a happy short story final for this story`
  })
  isGeneratingStoryPart.value = true
  isLoadingStory.value = true
  const responseStoryEnding = await getStoryPart()
  isGeneratingStoryPart.value = false
  isGeneratingImage.value = true
  const img = await createImage(responseStoryEnding)
  isLoadingStory.value = false
  await addMessage('ai', responseStoryEnding, null, null, img)

  //TODO: Disable chat form and show create new history button
}

function setLanguage(language: string) {
  preferredLanguage.value = language
  isLanguageSectionVisible.value = false
  startStoryTelling()
}
</script>

<template>
  <main>
    <section v-if="isLanguageSectionVisible" class="language-selection">
      <div class="icon-container" @click="setLanguage('english')">
        <img class="icon-language" :src="englishIcon" alt="English Language" />
        <span>Hello!</span>
      </div>
      <div class="icon-container" @click="setLanguage('spanish')">
        <img class="icon-language" :src="spanishIcon" alt="Spanish Language" />
        <span>Hola!</span>
      </div>
    </section>
    <section class="chat">
      <div
        class="chat-bubble"
        :class="{ ai: message.sender === 'ai' }"
        v-for="message in visibleMessages"
        :key="message.id"
      >
        <img
          class="message-image"
          v-if="message.image"
          :src="message.image.src"
          :alt="message.image.alt"
        />
        <div class="message-content">
          <img
            class="message-icon-pre"
            v-if="message.iconBack"
            :src="message.iconBack.path"
            :alt="message.iconBack.alt"
          />
          <p v-html="message.content"></p>
          <img
            class="message-icon"
            v-if="message.iconFront"
            :src="message.iconFront.path"
            :alt="message.iconFront.alt"
          />
        </div>
      </div>
      <div v-if="isLoadingStory" class="chat-bubble chat-bubble-loading">
        <div class="message-content message-content-loading">
          <p v-if="isGeneratingStoryPart">
            ✨ {{ preferredLanguage === 'english' ? 'Generating Story' : 'Generando Historia'
            }}<span class="dot">.</span><span class="dot">.</span><span class="dot">.</span>
          </p>
          <p v-if="!isGeneratingStoryPart">
            ✨
            {{ preferredLanguage === 'english' ? 'Generating Story' : 'Generando Historia' }}: 👍
          </p>
          <p v-if="isGeneratingImage">
            ✨ {{ preferredLanguage === 'english' ? 'Generating Image' : 'Generando Imagen'
            }}<span class="dot">.</span><span class="dot">.</span><span class="dot">.</span>
          </p>
        </div>
      </div>
    </section>
    <section v-if="isChatSectionVisible" class="chat-entry">
      <form class="chat-input" @submit.prevent="handleSubmit">
        <label for="message">{{
          preferredLanguage === 'english'
            ? 'WRITE THE NEXT STORY PART:'
            : 'ESCRIBE LA SIGUIENTE PARTE DE LA HISTORIA'
        }}</label>
        <textarea
          :disabled="!isChatFormEnabled"
          v-model="userInput"
          id="message"
          name="message"
          rows="3"
        ></textarea>
        <button :disabled="!isChatFormEnabled || !userInput.length" class="btn" type="submit">
          {{ preferredLanguage === 'english' ? 'SEND' : 'ENVIAR' }}
        </button>
        <button
          :disabled="!isChatFormEnabled || !isEndStoryButtonEnabled"
          class="btn"
          type="button"
          @click="endStory"
        >
          {{ preferredLanguage === 'english' ? 'END STORY' : 'FINALIZAR HISTORIA' }}
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

p {
  font-weight: bold;
}

main {
  display: flex;
  flex-direction: column;
  gap: 40px;
  margin: 20px;
}

.chat {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.language-selection {
  display: flex;
  justify-content: center;
  gap: 20px;
}

.icon-language {
  height: 50px;
}

.icon-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  cursor: pointer;
  transition: all 0.5s ease-in-out;
}

.icon-container:hover {
  transform: scale(1.2);
}

.chat-bubble {
  animation: slideIn 0.5s ease forwards;

  display: flex;
  flex-direction: column;
  align-items: center;
  width: 600px;
  /*margin: auto;*/
  position: relative;
  padding: 1rem;
  box-sizing: border-box;

  color: #000;
  background-clip: padding-box;
  border: solid 5px transparent;
  border-radius: 1em;
}

.chat-bubble::before {
  animation: slideIn 0.5s ease forwards;
  content: '';
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  z-index: -1;
  margin: -5px;
  border-radius: inherit;
  background: linear-gradient(to right, #ddedff, orange);
}

.chat-bubble.chat-bubble-loading::before {
  background: #fff;
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

.message-content {
  display: flex;
  gap: 10px;
  width: 100%;
}

.message-content-loading {
  flex-direction: column;
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

.dot {
  animation-name: dot;
  animation-duration: 1.3s;
  animation-iteration-count: infinite;
  animation-fill-mode: both;
  font-weight: 900;
}

.dot:nth-child(1) {
  animation-delay: 0.5s;
}

.dot:nth-child(2) {
  animation-delay: 0.7s;
}

.dot:nth-child(3) {
  animation-delay: 0.9s;
}

@keyframes dot {
  0%,
  80%,
  100% {
    opacity: 0;
  }
  40% {
    opacity: 1;
  }
}
</style>
