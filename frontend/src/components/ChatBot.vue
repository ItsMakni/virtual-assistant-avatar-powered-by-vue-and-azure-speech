<template>
    <div class="chat-widget">
      <!-- Minimized View -->
      <div v-if="!expanded" class="widget-minimized" @click="toggleChat">
        <img
          style="padding: 6px; background: white; border-radius: 0 16px 16px 16px;"
          src="@/assets/Hamma.jpg"
          alt="Avatar"
          class="avatar-icon"
        />
      </div>
  
      <!-- Expanded View -->
      <div v-else class="widget-expanded">
        <div class="chat-header">
          <button @click="toggleChat" class="close-btn">X</button>
          <div class="placeholder-image">
            <img v-if="talking" src="@/assets/talk.gif" alt="Viseme Image" class="viseme-image" />
            <img v-else src="@/assets/fix.gif" alt="Default Viseme Image" class="viseme-image" />
        </div>
                  <button @click="toggleMute" class="mute-btn">
            {{ muted ? 'Unmute' : 'Mute' }}
          </button>
        </div>
        <div class="chat-body">
          <div style="    height: 180px;
    overflow-y: auto;" class="messages" ref="messagesContainer">
            <p class="message ai">Hello, How can I help you today?</p>
            <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.sender]" :style="messageStyles(msg)">
              <p>{{ msg.text }}</p>
              <span class="timestamp">{{ msg.time }}</span>
            </div>
          </div>
        </div>
        <div class="chat-inputbar-container">
          <form @submit.prevent="sendMessage" class="input-form">
            <div class="">
                <label>
        <input type="checkbox" v-model="isExplanation" />
             Read Mode
        </label>



            </div>
            <div style="display: flex">
            <input style="padding: 1px;"
              type="text"
              v-model="userInput"
              placeholder="Type your message"
              required
              :disabled="isLoading"
            />
            <button type="submit" :disabled="isLoading">Send</button>
            <button style="display: flex; background: #ececec; color: black;" type="button" class="translator-icon">
  <img style="height: 20px;" src="@/assets/hands.png" alt="Sign Language" class="viseme-image" />
  <span class="tooltip">Sign Language Translator</span>
</button>

        </div>
          </form>
          <div v-if="isLoading" class="loading-spinner">Loading...</div>
          <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
        </div>
        <p class="disclaimer">A virtual assistant may make mistakes.</p>
      </div>
    </div>
  </template>
  
  <script>
import axios from 'axios';
export default {
  name: 'ChatWidget',
  data() {
    return {
      expanded: false, // Widget state: minimized/expanded
      muted: false, // Mute state
      messageToSend: '',
      isExplanation: false,
      userInput: '', // Holds the user's input
      messages: [], // Messages array
      isLoading: false, // Loading state for the bot
      errorMessage: '', // Error message for failed requests
      audioCache: {}, // Caching for audio files
      visemes: [], // Store viseme data
      currentVisemeImage: '',
      talking: false,      // Current viseme image path
      visemeTimeouts: [], // Timeouts for viseme synchronization
    };
  },
  computed: {
    // Determine messageType based on the toggle switch
    messageType() {
      return this.isExplanation ? 'explanation' : 'text';
    },
  },
  methods: {
    toggleChat() {
      this.expanded = !this.expanded;
    },
    toggleMute() {
      this.muted = !this.muted;
    },
    messageStyles(msg) {
      return {
        textAlign: msg.sender === 'user' ? 'right' : 'left',
        backgroundColor: msg.sender === 'user' ? '#d1e7dd' : '#f8d7da',
      };
    },
    async sendMessage() {
      if (!this.userInput.trim()) return;

      // Add user message
      const userMessage = {
        text: this.userInput,
        sender: 'user',
        time: new Date().toLocaleTimeString(),
      };
      this.messages.push(userMessage);

      // Scroll to latest message
      this.$nextTick(() => {
        const container = this.$refs.messagesContainer;
        container.scrollTop = container.scrollHeight;
      });

      // Clear input
      const messageToSend = this.userInput;
      this.userInput = '';
      this.isLoading = true;
      this.errorMessage = '';

      try {
        // Send message to backend
        const response = await axios.post('http://localhost:4000/api/chat', {
          message: messageToSend, 
          type: this.messageType, 
        });

        const aiResponse = response.data.response;
        const audioFilename = response.data.audio;
        const visemes = response.data.visemes; // Capture viseme data

        // Store viseme data
        this.visemes = visemes;
        this.currentVisemeImage = '';
        this.visemeTimeouts.forEach(timeout => clearTimeout(timeout));
        this.visemeTimeouts = [];

        // Add AI message
        const aiMessage = {
          text: aiResponse,
          sender: 'ai',
          time: new Date().toLocaleTimeString(),
        };
        this.messages.push(aiMessage);

        // Scroll to latest message
        this.$nextTick(() => {
          const container = this.$refs.messagesContainer;
          container.scrollTop = container.scrollHeight;
        });

        // Fetch and play audio with viseme synchronization
        this.playAudio(audioFilename);
      } catch (error) {
        console.error('Error sending message:', error);
        const errorMsg = 'Sorry, something went wrong. Please try again.';
        const errorMessage = {
          text: errorMsg,
          sender: 'ai',
          time: new Date().toLocaleTimeString(),
        };
        this.messages.push(errorMessage);
        this.errorMessage = errorMsg;
      } finally {
        this.isLoading = false;
      }
    },
    async playAudio(filename) {
  if (this.audioCache[filename]) {
    const cachedAudio = this.audioCache[filename];
    cachedAudio.play().catch((error) => {
      console.error('Error playing cached audio:', error);
    });
    return;
  }

  try {
    const audioResponse = await axios.get(`http://localhost:4000/api/audio/${filename}`, {
      responseType: 'blob',
    });

    const extension = filename.split('.').pop().toLowerCase();
    const mimeType = {
      mp3: 'audio/mpeg',
      wav: 'audio/wav',
      ogg: 'audio/ogg',
    }[extension] || 'audio/mpeg';

    const audioURL = window.URL.createObjectURL(new Blob([audioResponse.data], { type: mimeType }));
    const audio = new Audio(audioURL);
    this.audioCache[filename] = audio;

    // Set `talking` to true when audio starts
    audio.addEventListener('play', () => {
      this.talking = true;
    });

    // Set `talking` to false when audio ends
    audio.addEventListener('ended', () => {
      this.talking = false;
    });

    audio.play().catch((error) => {
      console.error('Error playing audio:', error);
    });
  } catch (error) {
    console.error('Error fetching or playing audio:', error);
  }
}

  },
  beforeDestroy() {
    // Clear any viseme timeouts
    this.visemeTimeouts.forEach(timeout => clearTimeout(timeout));
    this.visemeTimeouts = [];
  },
};

</script>

  
<style scoped>
/* General styles */
.chat-widget {
  position: fixed;
  background-image: url("../assets/Screenshot.png");
  bottom: 20px;
  right: 20px;
  z-index: 9999;
  font-family: Arial, sans-serif;
}

/* Minimized View */
.widget-minimized {
  cursor: pointer;
  width: 150px;
  height: 150px;
  border-radius: 0 16px 16px 16px;
  overflow: hidden;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.avatar-icon {
  width: 100%;
  height: 100%;
}
/* Tooltip container */
.translator-icon {
  position: relative;
  display: flex;
  background: #ececec;
  color: black;
  border: none;
  padding: 10px;
  cursor: pointer;
}

/* Tooltip text */
.tooltip {
  visibility: hidden;
  width: 120px; /* Adjusted width for smaller font */
  background-color: rgba(0, 0, 0, 0.7);
  color: #fff;
  text-align: center;
  border-radius: 5px;
  padding: 5px;
  position: absolute;
  z-index: 1;
  bottom: 125%; /* Position above the button */
  left: 50%;
  margin-left: -60px; /* Adjusted to center with smaller width */
  opacity: 0;
  transition: opacity 0.3s ease-in-out;
  font-size: 10px; /* Smaller font size */
}

/* Tooltip arrow */
.tooltip::after {
  content: "";
  position: absolute;
  top: 100%;
  left: 50%;
  margin-left: -5px;
  border-width: 5px;
  border-style: solid;
  border-color: rgba(0, 0, 0, 0.7) transparent transparent transparent;
}

/* Show the tooltip when hovering over the button */
.translator-icon:hover .tooltip {
  visibility: visible;
  opacity: 1;
}

/* Expanded View */
.widget-expanded {
  background: white;
  width: 320px;
  border-radius: 10px;
  box-shadow: 0 8px 15px rgba(0, 0, 0, 0.2);
  overflow: hidden;
}

.chat-inputbar-container {
    box-shadow: 0px -4px 16px 0px rgba(27, 59, 226, 0.08);
    padding: 7px;
}

.chat-header {
    background: #efefed;
    color: white;
    padding: 0px 20px;
    display: flex
;
    align-items: center;
    justify-content: space-between;
    height: 221px;
}

.avatar-image {
  width: 50px;
  height: 50px;
  border-radius: 50%;
}

.close-btn,
.mute-btn {
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  font-weight: bold;
}

.message {
  padding: 8px;
  border-radius: 5px;
  margin: 5px 0;
  font-size: 14px;
  max-width: 80%;
  position: relative;
}

.message.user {
  background-color: #d1e7dd;
  align-self: flex-end;
}

.message.ai {
  background-color: #f8d7da;
  align-self: flex-start;
}

.timestamp {
  display: block;
  font-size: 0.8rem;
  color: #666;
  margin-top: 0.2rem;
}

.chat-body {
    flex: 1;
    display: flex
;
    flex-direction: column-reverse;
    overflow-y: auto;
    padding: 8px 16px;
}

.messages {
  display: flex;
  flex-direction: column;
}

.input-form button {
  padding: 0.5rem 1rem;
  margin-left: 0.5rem;
  border: none;
  background-color: #0d6efd;
  color: white;
  border-radius: 4px;
  cursor: pointer;
}

.input-form button:disabled {
  background-color: #6c757d;
  cursor: not-allowed;
}

.input-form button:hover:not(:disabled) {
  background-color: #0b5ed7;
}

.loading-spinner {
  text-align: center;
  margin-top: 0.5rem;
  color: #555;
}

.error-message {
  color: red;
  text-align: center;
  margin-top: 0.5rem;
}

.disclaimer {
  font-size: 10px;
  color: gray;
  text-align: center;
  padding: 5px;
}
</style>
