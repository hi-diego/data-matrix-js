<template>
<div class="order-detector">
  <div class="d-flex">
  </div>
  <!-- <img :src="imageUrl" alt=""> -->
  <canvas ref="canvas" id="canvas" style="opacity: 1; position: fixed; bottom: 0; right: 0;"></canvas>
  <video id="video" playsinline autoplay style="opacity: 0.2; position: fixed; bottom: 0; right: 0;"></video>
  <p v-for="orderRaw in log.split('\n')" style="font-size: 3rem; font-weight: bold;"> {{ orderRaw }} </p>
</div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import axios from "axios";
import { sleep } from 'openai/core';

const imageUrl = ref(null);
const log = ref(null);
var loadingOpenAiApi = false;

const API_KEY = "";
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
          "text": "Detect the handwritten text in the image and return the plain text and nothing else, if you are not able to detect any text, return an empty string.",
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
    cv.cvtColor(opencvImage, opencvImage, cv.COLOR_RGBA2GRAY, 127);
    // low the resolution by half
    cv.pyrDown(opencvImage, opencvImage);
    // use ADAPTIVE_THRESH_GAUSSIAN_C 
    cv.adaptiveThreshold(opencvImage, opencvImage, 255, cv.ADAPTIVE_THRESH_GAUSSIAN_C, cv.THRESH_BINARY, 11, 2);
    // low the resolution by half
    // cv.pyrDown(opencvImage, opencvImage);
    

    cv.imshow(canvas, opencvImage);
    // convert opencvImage to base64
    const data = canvas.toDataURL();
    base64_image = data;
    // send to openai
    if (loadingOpenAiApi) return;
    log.value = 'sending...';
    loadingOpenAiApi = true;
    axios.post(url, payload(), { headers: headers })
      .then((response) => {
        log.value = response?.data?.choices[0]?.message?.content;
        loadingOpenAiApi = false;
      })
      .catch((error) => {
        console.log(error);
      });
}

var barcodeDetector = null; 
try {
  barcodeDetector = new BarcodeDetector({ formats: ['data_matrix', 'qr_code'] });
} catch (e) {
  console.log('BarcodeDetector is not supported by this browser.');
}



function detectBarcodes () {
  // log.value = 'detecting...';
  if (!barcodeDetector) {
    alert('BarcodeDetector is not supported by this browser.');
    return;
  }
  const imageData = captureCurrentFrame(video);
  if (!imageData) return;
  barcodeDetector.detect(imageData).then((_barcodes) => {
    if (_barcodes.length === 0) {
      // barcodes.value = 'No barcode detected.';
      return;
    }
    // log.value = 'Barcode detected!' + _barcodes.map(barcode => barcode.rawValue);
    // alert('Barcode detected!' + _barcodes.map(barcode => barcode.rawValue));
    navigator.vibrate(100); // vibrate for 200ms
    takePhoto();
  }).catch((error) => {
    alert(error);
  }).finally(() => {
    //
  });
}

function captureCurrentFrame(videoElement) {
  // Create a canvas with the same dimensions as the video.
  const _canvas = document.createElement('canvas');
  _canvas.width = videoElement.videoWidth;
  _canvas.height = videoElement.videoHeight;

  // Draw the current frame of the video onto the canvas.
  const ctx = _canvas.getContext('2d');
  ctx.drawImage(videoElement, 0, 0, _canvas.width, _canvas.height);

  // // Retrieve the image data from the canvas.
  // const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

  // return imageData;
  return _canvas;
}


onMounted(() => {
  canvas = canvas.value;
  video = document.getElementById('video');
  context = canvas.getContext('2d');
  // Initial camera setup  
  // switchCamera();
  // Take a photo every half second
  // setInterval(takePhoto, 5000);
  setInterval(detectBarcodes, 100);
});

</script>
