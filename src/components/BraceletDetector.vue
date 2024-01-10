<template lang="pug">
div()
  video(ref="videoElement" autoplay playsinline muted style="display: block; position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;")
  h1(style="position: relative; z-index: 1; display: block;") {{ userName }}
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as tf from "@tensorflow/tfjs";
import * as handpose from "@tensorflow-models/handpose";

const userName = ref(null);
const videoElement = ref(null);
const formats = ref([]);
const barcodes = ref([]);
let video, scene, camera, renderer, texture, material, geometry, mesh, composer;
const barcodeDetector = new BarcodeDetector({ formats: ['data_matrix'] });

onMounted(() => {
  initCamera();
});


function detectBarcodes () {
  formats.value = 'detecting...';
  // const create image from video
  const imageData = captureCurrentFrame(videoElement.value);
  const imgUrl = imageData.toDataURL();
  // rectangleImages.value.push(imgUrl);
  barcodeDetector.detect(imageData).then((_barcodes) => {
    if (_barcodes.length === 0) {
      barcodes.value = 'No barcode detected.';
      return;
    }
    barcodes.value = _barcodes.map(barcode => barcode.rawValue);
    navigator.vibrate(100); // vibrate for 200ms
    userName.value = barcodes.value[0];
  }).catch((error) => {
    barcodes.value = error;
    alert(error);
  }).finally(() => {
    // formats.value = 'detecting...';
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

function initCamera() {
  video = videoElement.value;
  if (navigator.mediaDevices.getUserMedia) {
    const constraints = {
      video: {
        width: { min: 1280 },
        height: { min: 720 },
        facingMode: 'user'
      }
    };
    navigator.mediaDevices.getUserMedia(constraints)
      .then((stream) => {
        video.srcObject = stream;
        video.play();
        window.setInterval(() => {
          // if (!!userName) return;
          detectBarcodes();
        }, 100);
      })
      .catch((error) => {
        console.log("Something went wrong!", error);
      });
  }
}
</script>

<style scoped>
</style>