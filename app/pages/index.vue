<script setup lang="ts">
import type { FormSubmitEvent } from '@nuxt/ui'
import * as v from 'valibot'
import HighHatClosed from '~/assets/sounds/HighHatClosed.mp3'
import HighHatOpen from '~/assets/sounds/HighHatOpen.mp3'

const Time = ['1/4', '2/4', '3/4', '4/4', '5/8', '6/8', '7/8', '8/8']
const Samples = ['High Hat Open', 'High Hat Closed', 'Custom']
const schema = v.object({
  bpm: v.pipe(v.number(), v.minValue(20, 'Invalid bpm'), v.maxValue(350, 'Invalid bpm')),
  timeSignature: v.picklist(Time),
  soundGeneral: v.blob(),
  soundAccent: v.blob(),
  beats: v.number(),
})

type Schema = v.InferOutput<typeof schema>

const state = reactive({
  bpm: 120,
  timeSignature: '4/4',
  soundGeneral: new Blob(),
  soundAccent: new Blob(),
  beats: 30,
})

const readFileAsArrayBuffer = (file: Blob) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.readAsArrayBuffer(file);
    reader.onload = () => resolve(reader.result);
    reader.onerror = () => reject(reader.error);
  });
};

const decodeAudioData = (arrayBuffer: ArrayBuffer) => {
  return new Promise((resolve, reject) => {
    const audioContext = new AudioContext();
    audioContext.decodeAudioData(arrayBuffer, resolve, reject);
  });
};

const readSoundGeneral = ref<AudioBuffer>();
const readSoundAccent = ref<AudioBuffer>();

function audioBufferToWav(buffer: AudioBuffer): ArrayBuffer {
  const numChannels = buffer.numberOfChannels;
  const sampleRate = buffer.sampleRate;
  const numFrames = buffer.length;
  const audioData = buffer.getChannelData(0); // Assuming mono

  const bytesPerSample = 2; // 16-bit WAV
  const blockAlign = numChannels * bytesPerSample;
  const byteRate = sampleRate * blockAlign;

  const dataSize = numFrames * blockAlign;
  const bufferSize = 44 + dataSize;

  const arrayBuffer = new ArrayBuffer(bufferSize);
  const dataView = new DataView(arrayBuffer);

  // --- WAV Header ---
  // RIFF identifier
  dataView.setUint32(0, 0x52494646, false); // "RIFF"
  // File size minus RIFF identifier and file size
  dataView.setUint32(4, bufferSize - 8, true);
  // Format
  dataView.setUint32(8, 0x57415645, false); // "WAVE"
  // Format chunk identifier
  dataView.setUint32(12, 0x666d7420, false); // "fmt "
  // Format chunk byte count
  dataView.setUint32(16, 16, true);
  // Format code (PCM = 1)
  dataView.setUint16(20, 1, true);
  // Channel number
  dataView.setUint16(22, numChannels, true);
  // Sample rate
  dataView.setUint32(24, sampleRate, true);
  // Byte rate (Sample Rate * Block Align)
  dataView.setUint32(28, byteRate, true);
  // Block align (Number of channels * Bytes per sample)
  dataView.setUint16(32, blockAlign, true);
  // Bits per sample
  dataView.setUint16(34, bytesPerSample * 8, true);
  // Data chunk identifier
  dataView.setUint32(36, 0x64617461, false); // "data"
  // Data chunk length
  dataView.setUint32(40, dataSize, true);

  // --- WAV Data ---
  for (let i = 0; i < numFrames; i++) {
    const sample = Math.max(-1, Math.min(1, audioData[i] ?? 1));
    const intSample = (sample < 0 ? sample * 0x8000 : sample * 0x7FFF) | 0;
    dataView.setInt16(44 + i * bytesPerSample, intSample, true);
  }

  return arrayBuffer;
}

const isGenerating = ref(false);
const errorMessage = ref('');
const downloadUrl = ref<string | null>(null);
const generateMetronome = async () => {
  errorMessage.value = '';
  downloadUrl.value = null;
  isGenerating.value = true;
  try {
    const sampleRate = readSoundGeneral.value?.sampleRate ?? 44100;
    const beatInterval = 60 / state.bpm;
    const totalDuration = beatInterval * state.beats;
    const offlineAudioContext = new OfflineAudioContext(1, sampleRate * totalDuration, sampleRate); // Mono
    const timeSignatureParts = state.timeSignature.split('/');
    const beatsPerMeasure = parseInt(timeSignatureParts[1] || '4');
    const remainder = parseInt(timeSignatureParts[0] || '0');
    const remainedM = remainder === 4 || remainder === 8 ? 0 : remainder - 1

    for (let i = 0; i < state.beats; i++) {
      const startTime = i * beatInterval;
      const bufferSource = offlineAudioContext.createBufferSource();

      if (i % beatsPerMeasure === remainedM) {
        bufferSource.buffer = readSoundAccent.value ?? null;
      } else {
        bufferSource.buffer = readSoundGeneral.value ?? null;
      }

      bufferSource.connect(offlineAudioContext.destination);
      bufferSource.start(startTime);
    }

    // --- Render Audio ---
    const renderedBuffer = await offlineAudioContext.startRendering();

    // --- Convert to WAV ---
    const wavData = audioBufferToWav(renderedBuffer);

    // --- Create Blob ---
    const wavBlob = new Blob([wavData], {type: 'audio/wav'});
    downloadUrl.value = URL.createObjectURL(wavBlob);
  } catch (error) {
    console.error('Error generating metronome:', error);
    errorMessage.value = 'Error generating metronome';
  } finally {
    isGenerating.value = false;
  }
};

async function onSubmit(event: FormSubmitEvent<Schema>) {
  await generateMetronome()
  if (downloadUrl.value) {
      const link = document.createElement('a');
      link.href = downloadUrl.value;
      link.download = 'metronome.wav';
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
  }
  console.log(event.data)
}

async function getFileGeneralObject(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (file) {
    const bufferGeneral = await readFileAsArrayBuffer(file)
    readSoundGeneral.value = await decodeAudioData(bufferGeneral as ArrayBuffer) as AudioBuffer
  }
}

async function getFileAccentObject(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (file) {
    const bufferAccent = await readFileAsArrayBuffer(file)
    readSoundAccent.value = await decodeAudioData(bufferAccent as ArrayBuffer) as AudioBuffer
  }
}
const generalChoice = ref<'High Hat Open' | 'High Hat Closed' | 'Custom'>('High Hat Closed')
const accentChoice = ref<'High Hat Open' | 'High Hat Closed' | 'Custom'>('High Hat Open')
async function assignSound(type: 'general' | 'accent') {
  if (generalChoice.value === 'Custom') return;
  const HighHatOpenFile = await fetch(HighHatOpen)
  const HighHatClosedFile = await fetch(HighHatClosed)
  const HighHatClosedBuffer = await HighHatClosedFile.arrayBuffer()
  const HighHatOpenBuffer = await HighHatOpenFile.arrayBuffer()
  if (type === 'general') {
    readSoundGeneral.value = await decodeAudioData(generalChoice.value === 'High Hat Closed' ? HighHatClosedBuffer : HighHatOpenBuffer) as AudioBuffer
  } else {
    readSoundAccent.value = await decodeAudioData(accentChoice.value === 'High Hat Closed' ? HighHatClosedBuffer : HighHatOpenBuffer) as AudioBuffer
  }
}
assignSound('general')
assignSound('accent')

const estimatedLength = computed(() => {
  if (!readSoundGeneral.value || !readSoundAccent.value) return 0;

  const beatInterval = 60 / state.bpm;
  const beatsPerMeasure = parseInt(state.timeSignature.split('/')[1] || '4');
  let maxFinishTime = 0;

  for (let i = 0; i < state.beats; i++) {
    const startTime = i * beatInterval;
    const sampleDuration =
      (i % beatsPerMeasure === 0)
        ? readSoundAccent.value.duration
        : readSoundGeneral.value.duration;
    const finishTime = startTime + sampleDuration;
    if (finishTime > maxFinishTime) {
      maxFinishTime = finishTime;
    }
  }
  return Math.round(maxFinishTime);
});


</script>

<template>
  <div class="flex flex-col items-center justify-center gap-4 h-screen w-screen relative">
    <div class="absolute inset-0 bg-grid animate-[pulse_10s_ease-in-out_infinite] delay-700 z-0"/>
    <div class="absolute top-5 right-5"><ColorModeButton  /></div>
    <div class="flex flex-col justify-center items-center gap-2 bg-(--ui-bg) p-10 shadow-2xl border-r-slate-500 rounded-2xl z-50 mx-10">
    <div class="flex gap-2">
      <h1 class="font-bold text-2xl text-(--ui-primary) underline">
      Clicky
    </h1>
    </div>
    <UForm
        :schema="v.safeParser(schema)"
        :state="state"
        class="space-y-4"
        @submit="onSubmit"
        @change="downloadUrl = ''"
    >
        <UFormField label="BPM" name="bpm">
          <UInputNumber v-model="state.bpm" :min="20" :max="350" class="w-full"/>
        </UFormField>
        <UFormField label="Time Signature" name="timeSignature">
          <USelect v-model="state.timeSignature" :items="Time" class="w-full"/>
        </UFormField>
        <p>Sound</p>
        <div class="flex w-full gap-2">
          <UFormField label="Accent" name="soundAccent">
            <USelect v-model="accentChoice" :items="Samples" class="w-full" @change="assignSound('accent')" />
            <br>
            <UInput v-if="accentChoice === 'Custom'" class="w-full pt-2" type="file" @change="getFileAccentObject"/>
          </UFormField>
          <UFormField label="General" name="soundGeneral">
            <USelect v-model="generalChoice" :items="Samples" class="w-full" @change="assignSound('general')" />
            <br>
            <UInput v-if="generalChoice === 'Custom'" class="w-full pt-2" type="file" @change="getFileGeneralObject"/>
          </UFormField>
        </div>
        <UFormField label="Beats" name="beats" :hint="'Estimated length: ' + estimatedLength + 's'">
          <UInputNumber v-model="state.beats" class="w-full"/>
        </UFormField>
        <UButton :disabled="isGenerating" type="submit" class="float-end" >
          {{ isGenerating ? "Generating..." : "Download" }}
        </UButton>
        <p class="text-red-500 font-bold text-md">{{ errorMessage }}</p>
      </UForm>
    </div>
    <div class="flex gap-2 (--ui-bg) items-center bg-primary-foreground z-50">
        <UButton variant="ghost" href="https://www.rzn-music.com/">
            <UAvatar size="2xl" alt="RZN Music" src="https://i1.sndcdn.com/avatars-MlzK5Cx7ITceG4Vb-cnyRLw-t500x500.jpg" />
        </UButton>
        <UButton variant="ghost" href="https://github.com/zzz-adev">
            <UAvatar size="3xl" alt="Sleepy Dev" src="https://avatars.githubusercontent.com/u/175128483?v=4&size=64" />
        </UButton>
    </div>
  </div>
</template>
<style lang="postcss" scoped>
.bg-grid {
  background-image: linear-gradient(to right, oklch(0.666 0.179 58.318) 1px, transparent 1px), linear-gradient(to bottom, oklch(0.666 0.179 58.318) 1px, transparent 1px);
  background-size: 60px 60px;
  opacity: 0.5;
}
</style>
