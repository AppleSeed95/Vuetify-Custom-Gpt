<template>
  <v-container
    style="height: 100vh"
    class="d-flex flex-column justify-space-between"
  >
    <v-row>
      <v-col cols="12">
        <div ref="messagesContainer" id="messages-container">
          <div class="d-flex" v-for="(msg, index) in messages">
            <span class="mr-4 text-h6">{{
              msg.role === "system" ? "✨" : "🧠"
            }}</span>
            <v-card
              :key="index"
              class="mb-4"
              :variant="msg.role === 'system' ? 'flat' : 'outlined'"
            >
              <v-card-text
                class="p-8"
                v-html="formatMessage(msg.content)"
              ></v-card-text>
            </v-card>
          </div>
        </div>
      </v-col>
    </v-row>

    <v-row
      v-if="messages.length === 0"
      class="justify-space-between flex-grow-0"
    >
      <v-spacer></v-spacer>
      <v-card
        :key="index"
        class="ma-4"
        variant="tonal"
        color="primary"
        @click="handleSubmitTemplatePrompt('What is Vuetify?')"
      >
        <v-card-text>What is Vuetify?</v-card-text>
      </v-card>
      <v-card
        :key="index"
        class="ma-4"
        variant="tonal"
        color="primary"
        @click="
          handleSubmitTemplatePrompt('Help me write simple Button component')
        "
      >
        <v-card-text>Help me write simple Button component</v-card-text>
      </v-card>
      <v-card
        :key="index"
        class="ma-4"
        variant="tonal"
        color="primary"
        @click="handleSubmitTemplatePrompt('Assist in a Vuetify task')"
      >
        <v-card-text>Assist in a Vuetify task</v-card-text>
      </v-card>
      <v-spacer></v-spacer>
    </v-row>

    <v-row class="flex-grow-0">
      <v-col cols="12">
        <v-textarea
          v-model="message"
          label="Enter your message"
          variant="outlined"
          @click:append="sendMessage"
          @keydown.enter.prevent="sendMessage"
          single-line
          rows="2"
          no-resize
        >
          <template v-slot:append>
            <v-btn
              @click="sendMessage"
              :disabled="asyncState.isLoading"
              icon="mdi-send"
              variant="plain"
            ></v-btn>
          </template>
        </v-textarea>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { ref, watch, nextTick, reactive } from "vue";
import { marked } from "marked";

const messagesContainer = ref(null);
const message = ref("");
const messages = ref([]);
const asyncState = reactive({ isLoading: false, error: null });

const openAIKey = import.meta.env.VITE_APP_OPENAI_KEY;
const endpoint = import.meta.env.VITE_APP_ENDPOINT;

async function fetchOpenAIResponse(userMessage) {
  const copyOfUserMessage = userMessage;
  asyncState.isLoading = true;
  const currentMessageIndex = messages.value.length;
  messages.value.push({ role: "system", content: "" });
  try {
    // const response = await fetch("https://api.openai.com/v1/chat/completions", {
    //   method: "POST",
    //   headers: {
    //     "Content-Type": "application/json",
    //     Authorization: `Bearer ${openAIKey}`,
    //   },
    //   body: JSON.stringify({
    //     model: "gpt-3.5-turbo",
    //     messages: [...messages.value, { role: "user", content: userMessage }],
    //     stream: true,
    //   }),
    // });
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        user_message: userMessage,
      }),
    });
    const reader = response.body.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      const decoded = new TextDecoder("utf-8").decode(value);
      const lines = decoded
        .replaceAll(/^data: /gm, "")
        .split("\n")
        .filter((c) => c);

      for (let line of lines) {
        if (line !== "[DONE]") {
          try {
            // const chunk = JSON.parse(line);
            const chunk = line;
            // if (
            //   chunk.choices &&
            //   chunk.choices[0] &&
            //   chunk.choices[0].delta &&
            //   chunk.choices[0].delta.content
            // ) {
            //   messages.value[currentMessageIndex].content +=
            //     chunk.choices[0].delta.content;
            //   console.log(chunk.choices[0].delta.content);
            // }
            if (chunk) {
              console.log(chunk);
              messages.value[currentMessageIndex].content += chunk;
            }
          } catch (error) {
            console.error("Error parsing JSON line:", error, line);
          }
        }
      }
    }
  } catch (error) {
    message.value = copyOfUserMessage;
    console.error("Error fetching OpenAI response:", error);
    return "Sorry, I couldn't process that.";
  } finally {
    asyncState.isLoading = false;
  }
}

async function sendMessage() {
  if (asyncState.isLoading) return;
  if (message.value.trim()) {
    messages.value.push({ role: "user", content: message.value });
    const userInput = message.value; // Save the user input before clearing the message
    message.value = "";

    const LlmResponse = await fetchOpenAIResponse(userInput);
  }
}

const handleSubmitTemplatePrompt = (template) => {
  message.value = template;
  sendMessage();
};

watch(
  messages,
  () => {
    //scroll to the bottom of the messages container
    nextTick(() => {
      if (messagesContainer.value) {
        messagesContainer.value.scrollTop =
          messagesContainer.value.scrollHeight;
      }
    });
  },
  { deep: true }
);

const formatMessage = (content) => {
  return marked(content);
};
</script>

<style>
#messages-container {
  max-height: 80vh;
  overflow-y: auto;
}
</style>
