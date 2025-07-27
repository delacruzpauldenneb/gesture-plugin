
<template>
  <div>
    <video ref="videoRef" autoplay playsinline class="mx-auto rounded shadow" width="640" height="480" />
    <p class="text-xl font-semibold mt-4">Detected Gesture: {{ gesture }}</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as handpose from '@tensorflow-models/handpose'
import '@tensorflow/tfjs-backend-webgl'

const videoRef = ref(null)
const gesture = ref('None')
let model = null
let intervalId = null

const detectGesture = (landmarks) => {
  if (!landmarks || landmarks.length === 0) return 'None'

  const fingers = {
    thumb: landmarks[4][1] < landmarks[3][1],
    index: landmarks[8][1] < landmarks[6][1],
    middle: landmarks[12][1] < landmarks[10][1],
    ring: landmarks[16][1] < landmarks[14][1],
    pinky: landmarks[20][1] < landmarks[18][1],
  }

  const openFingers = Object.values(fingers).filter(Boolean).length

  if (openFingers === 0) return 'Fist'
  if (openFingers === 5) return 'Open Hand'
  if (fingers.thumb && !fingers.index && !fingers.middle && !fingers.ring && !fingers.pinky) return 'Thumbs Up'

  return 'Unknown'
}

const setupCamera = async () => {
  const stream = await navigator.mediaDevices.getUserMedia({ video: true })
  videoRef.value.srcObject = stream

  return new Promise((resolve) => {
    videoRef.value.onloadedmetadata = () => {
      resolve(videoRef.value)
    }
  })
}

const runDetection = async () => {
  const predictions = await model.estimateHands(videoRef.value)
  if (predictions.length > 0) {
    gesture.value = detectGesture(predictions[0].landmarks)
  } else {
    gesture.value = 'None'
  }
}

onMounted(async () => {
  await setupCamera()
  model = await handpose.load()
  intervalId = setInterval(runDetection, 200)
})

onBeforeUnmount(() => {
  clearInterval(intervalId)
  const tracks = videoRef.value?.srcObject?.getTracks() || []
  tracks.forEach(track => track.stop())
})
</script>
