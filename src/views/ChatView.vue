<template>
  <div class="chat-container">
    <div class="header">
      <img src="/src/assets/seattlegrace.jpg" alt="Seattle Grace Hospital Logo" class="logo" />
      <h1>Seattle Grace Chat Assistant</h1>
    </div>
    <div class="chat-window" ref="chatWindow">
      <div v-for="(message, index) in messages" :key="index">
        <!-- Bot message with profile -->
        <div v-if="message.role === 'system'" class="bot-message-container">
          <img src="/src/assets/boticon.png" alt="Bot Profile" class="bot-profile" />
          <div class="bot-message">{{ message.content }}</div>
        </div>

        <!-- User message -->
        <div v-else class="user-message">
          {{ message.content }}
        </div>
      </div>
    </div>

    <div class="chat-input">
      <input
        type="text"
        v-model="userInput"
        placeholder="Type your query..."
        @keyup.enter="sendGenerateMessage"
      />
      <button @click="sendGenerateMessage">Send</button>
    </div>
  </div>
</template>

<script>
import ollama from 'ollama';
const CONTEXT_LIMIT = 9500;
export default {
  name: "ChatView",
  data() {
    return {
      messages: [],
      userInput: '',
      context: [] //de context is een array aan tokens, deze wordt opgeslagen bij een response en meegestuurd bij een nieuwe om context te geven aan de prompt
    };
  },
  methods: {
    async sendGenerateMessage() {
      if (this.userInput.trim() !== "")
      {
        //Messages wordt geupdate en de prompt wordt gedefinieerd.
        this.messages.push({ role: 'user', content: this.userInput });
        const prompt = this.userInput;
        this.userInput = '';

        // Toon een tijdelijk bericht tijdens het proberen van de API call
        this.messages.push({ role: 'system', content: 'thinking...' });
        this.scrollToBottom();

        try {
          console.log(prompt)
          const output = await ollama.generate({ model: "llama3.2", prompt: prompt, stream: true, context: this.context });


          //het antwoord wordt in losse tokens gestuurd om het effect van typen te geven
          let responseText = '';
          for await (const part of output) {

            //het antwoord wordt per token in de stream geupdate en getoond
            responseText += part.response;
            this.messages[this.messages.length - 1].content = responseText;
            this.scrollToBottom();

            //als het antwoord compleet is, wordt de context geupdate
            if (part.done === true) {
              console.log(part.context);
              this.context.push(...part.context);

              // Controleer of de contextlimiet is overschreden
              if (this.context.length > CONTEXT_LIMIT) {
                // Verwijder de oudste items uit de context om binnen de limiet te blijven
                const excess = this.context.length - CONTEXT_LIMIT;
                this.context.splice(0, excess);
              }
            }
          }

        } catch (error) {
          console.error('Fout bij het ophalen van antwoord:', error);
          this.messages.push({ role: 'system', content: "Sorry, something went wrong." });
          this.scrollToBottom();
        }
      }

    },
    scrollToBottom() {
      this.$nextTick(() => {
        const chatWindow = this.$refs.chatWindow;
        if (chatWindow) {
          chatWindow.scrollTop = chatWindow.scrollHeight;
        }
      });
    }
  }
}
</script>

<style scoped>
.chat-container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  background-color: #f4f6f9;
  color: #333;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.header {
  display: flex;
  align-items: center;
  flex-direction: column;
  margin-bottom: 20px;
  font-family: "Times New Roman";
}

.logo {
  width: 350px;
  height: 120px;
}

h1 {
  text-align: center;
  color: #003366;
}

.chat-window {
  border: 1px solid #d1d5db;
  border-radius: 5px;
  padding: 10px;
  height: 400px;
  overflow-y: scroll;
  background-color: #ffffff;
  margin-bottom: 10px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.bot-message-container {
  display: flex;
  align-items: flex-start;
  gap: 10px;
}

.bot-message {
  background-color: #dbdbdb;
  color: #000000;
  padding: 8px 12px;
  border-radius: 12px;
  width: fit-content;
  max-width: 80%;
  word-wrap: break-word;
}

.bot-profile {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background-color: #E9E9E9;
}

.user-message {
  text-align: right;
  background-color: #004488;
  color: #ffffff;
  padding: 8px 12px;
  border-radius: 12px;
  width: fit-content;
  max-width: 80%;
  margin-left: auto;
  word-wrap: break-word;
}

.chat-input {
  display: flex;
  gap: 10px;
}

.chat-input input {
  flex: 1;
  padding: 12px;
  border: 1px solid #d1d5db;
  border-radius: 5px;
  background-color: #ffffff;
  color: #333;
}

.chat-input input::placeholder {
  color: #999;
}

.chat-input button {
  padding: 10px 20px;
  background-color: #003366;
  color: #ffffff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.chat-input button:hover {
  background-color: #002244;
}
</style>
