<template>
<div class="order-detector">
  <div class="d-flex">
  </div>
  <pre> {{ log }} </pre>
  <!-- <img :src="imageUrl" alt=""> -->
  <canvas ref="canvas" id="canvas" style="opacity: 1; position: absolute; top: 0; left 0;"></canvas>
  <video id="video" playsinline autoplay></video>
</div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import axios from "axios";
import { sleep } from 'openai/core';

const imageUrl = ref(null);
const log = ref(null);

const API_KEY = "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";
const url = "https://api.openai.com/v1/chat/completions";
var base64_image = null;

const headers = {
  "Content-Type": "application/json",
  "Authorization": "Bearer " + API_KEY,
};

const payload = () => ({
  "model": "gpt-4-vision-preview",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Detect the handwritten text in the image and return the raw text and nothing more",
        },
        {
          "type": "image_url",
          "image_url": {
            "url": base64_image
          }
        }
      ]
    }
  ],
  "max_tokens": 300
})

var video = null;
var canvas = ref('canvas');
var context = null;
var useFrontCamera = false;
var mainStream = null;

if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
  const constraints = {
    video: {
      facingMode: "environment"
    }
  };
  navigator.mediaDevices.getUserMedia(constraints)
    .then(function(stream) {
      video.srcObject = stream;
      video.play();
      mainStream = stream;
    });
}

// Function to take a photo
function takePhoto() {
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    context.drawImage(video, 0, 0, canvas.width, canvas.height);
    // binarize image using opencv js
    const opencvImage = cv.imread(canvas);
    cv.cvtColor(opencvImage, opencvImage, cv.COLOR_RGBA2GRAY, 100);
    // use ADAPTIVE_THRESH_GAUSSIAN_C 
    cv.adaptiveThreshold(opencvImage, opencvImage, 255, cv.ADAPTIVE_THRESH_GAUSSIAN_C, cv.THRESH_BINARY, 11, 2);
    // low the resolution by half
    cv.pyrDown(opencvImage, opencvImage);
    cv.pyrDown(opencvImage, opencvImage);

    cv.imshow(canvas, opencvImage);
    // convert opencvImage to base64
    const data = canvas.toDataURL();
    base64_image = data;
    // send to openai
    axios.post(url, payload(), { headers: headers })
      .then((response) => {
        log.value = response?.data?.choices?.message?.content;
      })
      .catch((error) => {
        console.log(error);
      });
}

onMounted(() => {
  canvas = canvas.value;
  video = document.getElementById('video');
  context = canvas.getContext('2d');
  // Initial camera setup  
  // switchCamera();
  // Take a photo every half second
  setInterval(takePhoto, 2000);
});

</script>
