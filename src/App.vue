<script setup>
import { reactive, ref } from 'vue'

async function createChatCompletion(question, onMessage) {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${import.meta.env.VITE_OPENAI_API_KEY}`,
    },
    method: 'POST',
    body: JSON.stringify({
      model: 'gpt-3.5-turbo',
      stream: true,
      messages: [{ role: 'user', content: question }],
    }),
  })

  const reader = response.body.getReader()
  reader.read().then(function processText({ done, value }) {
    if (done) {
      return
    }

    const data = new TextDecoder('utf-8').decode(value)

    const lines = data
      .toString()
      .split('\n')
      .filter((line) => line.trim() !== '')

    for (const line of lines) {
      const message = line.replace(/^data: /, '')
      if (message === '[DONE]') {
        return // Stream finished
      }
      try {
        const parsed = JSON.parse(message)
        const content = parsed.choices[0].delta.content

        if (content && content !== '\n\n') {
          onMessage(content)
        }
      } catch (error) {
        console.error('Could not JSON parse stream message', message, error)
      }
    }

    return reader.read().then(processText)
  })
}

const inputEl = ref(null)
const state = reactive({
  messages: [],
})

async function handleSubmit() {
  const question = inputEl.value.textContent
  state.messages.push(question)
  inputEl.value.textContent = ''
  state.messages.push('')

  await createChatCompletion(question, (message) => {
    state.messages[state.messages.length - 1] += message
  })
}
</script>

<template>
  <div class="w-9/12 mx-auto">
    <form class="flex justify-center items-start gap-2">
      <span
        contenteditable
        ref="inputEl"
        @keydown.meta.enter="handleSubmit"
        class="border-solid border-2 border-black w-full inline-block leading-10 px-2"
      />
      <button
        @click.prevent="handleSubmit"
        class="border-solid border-2 border-black px-2 leading-10"
      >
        submit
      </button>
    </form>

    <div class="mt-4 whitespace-pre-wrap">
      <div class="mt-4" v-for="message in state.messages" :key="message">
        {{ message }}
      </div>
    </div>
  </div>
</template>
