<template>
  <div class="flex flex-col items-center justify-center min-h-screen bg-slate-800 text-white">
    <div class="w-full max-w-md p-8 space-y-4 bg-slate-700 rounded-lg">
      <div class="relative">
        <textarea
          v-model="text"
          :disabled="isTiming"
          class="w-full p-4 text-lg bg-slate-600 border-2 border-slate-500 rounded-lg resize-none focus:outline-none focus:border-purple-500"
          rows="5"
          placeholder="Enter text here..."
        ></textarea>
        <div class="absolute top-2 right-2 text-sm text-slate-400">{{ wordCount }} words</div>
      </div>

      <div v-if="showWarning" class="text-red-400 text-sm">
        Please enter some text to start.
      </div>

      <div v-if="!isTiming && !isStopped" class="flex space-x-4">
        <button
          @click="start"
          class="w-full py-2 text-lg font-bold text-white bg-purple-600 rounded-lg hover:bg-purple-700 focus:outline-none"
        >
          Start
        </button>
        <button
          @click="generateText"
          class="w-full py-2 text-lg font-bold text-white bg-teal-500 rounded-lg hover:bg-teal-600 focus:outline-none"
        >
          Generate
        </button>
      </div>

      <div v-if="isTiming" class="flex flex-col items-center space-y-4">
        <div class="text-4xl font-bold">{{ formattedTime }}</div>
        <button
          @click="stop"
          class="w-full py-2 text-lg font-bold text-white bg-rose-500 rounded-lg hover:bg-rose-600 focus:outline-none"
        >
          Stop
        </button>
      </div>

      <div v-if="isStopped" class="flex flex-col items-center space-y-4">
        <div class="text-4xl font-bold">{{ formattedTime }}</div>
        <div class="flex space-x-4">
          <button
            @click="save"
            class="w-full px-8 py-2 text-lg font-bold text-white bg-emerald-600 rounded-lg hover:bg-emerald-700 focus:outline-none"
          >
            Save
          </button>
          <button
            @click="reset"
            class="w-full px-8 py-2 text-lg font-bold text-white bg-slate-600 rounded-lg hover:bg-slate-700 focus:outline-none"
          >
            New
          </button>
        </div>
      </div>

      <div v-if="(isTiming || isStopped) && wordsToSpeak.length > 0" class="flex flex-wrap justify-center gap-2 mt-4">
<span v-for="(word, index) in wordsToSpeak" :key="index" class="px-3 py-1 rounded-full text-sm" :class="isStopped && wordsToSpeak.length > 0 ? 'bg-rose-400' : 'bg-slate-600'">
          {{ word }}
        </span>
      </div>

      <div v-if="lastPronouncedText" class="text-center text-lg my-2">
        <p>You said: <span class="font-bold">{{ lastPronouncedText }}</span></p>
      </div>

      <div v-if="isTiming || isStopped" class="text-center text-lg">
        <p>Pronounced Words: {{ pronouncedWordsCount }} / {{ wordCount }}</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from 'vue';

// Component State
const text = ref(''); // The text to be spoken
const isTiming = ref(false); // True when the timer is running
const isStopped = ref(false); // True when the timer is stopped
const showWarning = ref(false); // True when a warning message should be shown
const timer = ref(null); // The timer instance
const time = ref(0); // The current time in seconds
const recognition = ref(null); // The speech recognition instance
const pronouncedWords = ref([]); // The words that have been pronounced
const wordsToSpeak = ref([]); // The words that are left to be pronounced
const lastPronouncedText = ref(''); // The last pronounced text
let lastTranscript = ''; // The last transcript from the speech recognition

// Computed Properties

// The total number of words in the text
const wordCount = computed(() => {
  return text.value.trim() === '' ? 0 : text.value.trim().split(/\s+/).length;
});

// The number of words that have been pronounced
const pronouncedWordsCount = computed(() => {
  return pronouncedWords.value.length;
});

// The formatted time string (MM:SS)
const formattedTime = computed(() => {
  const minutes = Math.floor(time.value / 60).toString().padStart(2, '0');
  const seconds = (time.value % 60).toString().padStart(2, '0');
  return `${minutes}:${seconds}`;
});

// Methods

/**
 * Removes punctuation from a word and converts it to lowercase.
 * @param {string} word The word to clean.
 * @returns {string} The cleaned word.
 */
const cleanWord = (word) => {
  return word.replace(/[.,\/#!$%\^&\*;:{}=\-_`~()?]/g, "").toLowerCase();
}

/**
 * Starts the timer and speech recognition.
 */
const start = () => {
  if (text.value.trim() === '') {
    showWarning.value = true;
    return;
  }
  showWarning.value = false;
  isTiming.value = true;
  isStopped.value = false;
  time.value = 0;
  pronouncedWords.value = [];
  wordsToSpeak.value = text.value.trim().split(/\s+/).map(word => cleanWord(word));
  lastTranscript = '';
  lastPronouncedText.value = '';
  timer.value = setInterval(() => {
    time.value++;
  }, 1000);

  startSpeechRecognition();
};

/**
 * Stops the timer and speech recognition.
 */
const stop = () => {
  isTiming.value = false;
  isStopped.value = true;
  clearInterval(timer.value);
  stopSpeechRecognition();
};

/**
 * Saves the current state (not implemented).
 */
const save = () => {
  const svgContent = `
    <svg width="400" height="300" xmlns="http://www.w3.org/2000/svg">
      <rect width="100%" height="100%" fill="#1f2937" />
      <text x="10" y="30" font-family="Arial" font-size="16" fill="#ffffff">
        Original Text:
      </text>
      <text x="10" y="50" font-family="Arial" font-size="14" fill="#ffffff">
        ${text.value}
      </text>
      <text x="10" y="100" font-family="Arial" font-size="20" fill="#ffffff">
        Time: ${formattedTime.value}
      </text>
      <text x="10" y="130" font-family="Arial" font-size="16" fill="#ffffff">
        Pronounced Words: ${pronouncedWordsCount.value} / ${wordCount.value}
      </text>
    </svg>
  `;

  const blob = new Blob([svgContent], { type: 'image/svg+xml;charset=utf-8' });
  const url = URL.createObjectURL(blob);

  const img = new Image();
  img.onload = () => {
    const canvas = document.createElement('canvas');
    canvas.width = 400;
    canvas.height = 300;
    const ctx = canvas.getContext('2d');
    ctx.drawImage(img, 0, 0);
    URL.revokeObjectURL(url);

    const pngUrl = canvas.toDataURL('image/png');

    const a = document.createElement('a');
    a.href = pngUrl;
    a.download = 'whispers-results.png';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
  };

  img.src = url;
};

/**
 * Resets the component to its initial state.
 */
const reset = () => {
  isTiming.value = false;
  isStopped.value = false;
  showWarning.value = false;
  time.value = 0;
  text.value = '';
  pronouncedWords.value = [];
  wordsToSpeak.value = [];
  lastPronouncedText.value = '';
};

/**
 * Generates a random text if the textarea is empty.
 */
const generateText = async () => {
  try {
    const response = await fetch('https://dummyjson.com/quotes/random');
    const data = await response.json();
    text.value = data.quote;
    showWarning.value = false;
  } catch (error) {
    console.error('Error fetching quote:', error);
    text.value = 'Failed to fetch a quote. Please try again.';
  }
};

/**
 * Initializes and starts the speech recognition.
 */
const startSpeechRecognition = () => {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    console.error('Speech recognition not supported in this browser.');
    return;
  }

  recognition.value = new SpeechRecognition();
  recognition.value.continuous = true;
  recognition.value.interimResults = true;
  recognition.value.lang = 'en-US';

  /**
   * This event is fired when the speech recognition service returns a result.
   * @param {SpeechRecognitionEvent} event The event containing the recognition results.
   */
  recognition.value.onresult = (event) => {
    let interimTranscript = '';
    let finalTranscript = '';

    for (let i = 0; i < event.results.length; i++) {
      const transcript = event.results[i][0].transcript;
      if (event.results[i].isFinal) {
        finalTranscript += transcript + ' ';
      } else {
        interimTranscript += transcript;
      }
    }

    lastPronouncedText.value = interimTranscript;

    if (finalTranscript) {
      const newWords = finalTranscript.trim().split(/\s+/).map(word => cleanWord(word));
      
      newWords.forEach(word => {
        const index = wordsToSpeak.value.indexOf(word);
        if (index !== -1) {
          pronouncedWords.value.push(word);
          wordsToSpeak.value.splice(index, 1);
        }
      });

      if (wordsToSpeak.value.length === 0) {
        stop();
      }
    }
  };

  recognition.value.start();
};

/**
 * Stops the speech recognition.
 */
const stopSpeechRecognition = () => {
  if (recognition.value) {
    recognition.value.stop();
  }
};

// Cleanup when the component is unmounted
onUnmounted(() => {
  clearInterval(timer.value);
  stopSpeechRecognition();
});
</script>
