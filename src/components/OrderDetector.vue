<template>
<div class="order-detector">
  <div class="d-flex">
    <!-- <img :src="imageUrl" alt="" height="240" width="320"> -->
  </div>
  <pre> {{ log }} </pre>
  <video id="video" playsinline autoplay></video>
  <canvas id="canvas" style="opacity: 0;"></canvas>
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
const base64_image = null;

const headers = {
  "Content-Type": "application/json",
  "Authorization": "Bearer " + API_KEY,
};

const payload = {
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
};

var video = null;
var canvas = null;
var context = null;
var useFrontCamera = false;
var mainStream = null;


        
// Function to switch camera
function switchCamera() {
  // if (mainStream) mainStream.getTracks().forEach(track => track.stop());
  // useFrontCamera = !useFrontCamera;
  log.value = "Switching camera to " + (useFrontCamera ? "front" : "back");
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        let constraints = {
            video: {
                facingMode: (useFrontCamera ? "user" : "environment")
            }
        };
        navigator.mediaDevices.getUserMedia(constraints).then(function(stream) {
          mainStream = stream;
          video.srcObject = null;
          video.srcObject = stream;
          video.play();
          sleep(500).then(() => {
            takePhoto();
            // Send the image to OpenAI
            // axios.post(url, payload, { headers: headers }).then(function(response) {
            //     console.log(response.data);
            //     imageUrl.value = response.data.choices[0].text;
            // }).catch(function(error) {
            //     console.log("Error: " + error);
            // });
          });

        }).catch(function(error) {
            console.log("Error: " + error);
        });
    }
}

// Function to take a photo
function takePhoto() {
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    context.drawImage(video, 0, 0, canvas.width, canvas.height);
    // Here you can use canvas.toDataURL() to get the image data
    console.log(canvas.toDataURL());
    // Switch camera for the next shot
    switchCamera();
}

onMounted(() => {
  video = document.getElementById('video');
  canvas = document.getElementById('canvas');
  context = canvas.getContext('2d');
  // Initial camera setup  
  switchCamera();
});

</script>
