<template lang="pug">
.app()
  video(ref="videoElement" autoplay playsinline muted style="display: block; position: absolute; top: 0; left: 0; width: 100vw; height: 100vh; filter: grayscale(1)")
  h1(style="position: relative; z-index: 4; display: block; color: white;") {{ userName === null ? 'Escanea tu pulsera' : userName }}
  h1(style="position: relative; z-index: 1; display: block;") {{ controlsY }}
  AppMenu(v-if="userName" :items="categories" :scroll="controlsY")
</template>

<script setup>
import { ref, onMounted } from 'vue';
import AppMenu from './AppMenu.vue';
import * as tf from "@tensorflow/tfjs";
import * as handpose from "@tensorflow-models/handpose";

const userName = ref(null);
const videoElement = ref(null);
const formats = ref([]);
const barcodes = ref([]);
const controlsY = ref(0);
let video, scene, camera, renderer, texture, material, geometry, mesh, composer;
var barcodeDetector = null;
const categories = ['Cafe', 'Entradas', 'Cerveza', 'Vino', 'Cocktails', 'Destilados', 'Café', 'Té', 'Postres'];
var net = null;

const loadHandModel = async () => {
  console.log('Loading Handpose model...');
  net = await handpose.load();
  console.log('Handpose model loaded.');
  return net;
};

try {
  barcodeDetector = new BarcodeDetector({ formats: ['data_matrix'] });
} catch (e) {
  console.log('BarcodeDetector is not supported by this browser.');
}

onMounted(async () => {
  loadHandModel();
  initCamera();
});


function detectBarcodes () {
  if (!barcodeDetector) {
    // alert('BarcodeDetector is not supported by this browser.');
    return;
  }
  if (userName.value !== null) return;
  if (!videoElement.value) return;
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

const detectHand = async (network, _video) => {
  // Make Detections
  // if (0.9 < hand && hand[0] && hand[0].handInViewConfidence) return;
  const hand = await network.estimateHands(_video);
  if (!(hand && hand[0])) return;
  // Draw mesh
  const palmBaseXCoordinate = hand[0]?.annotations?.palmBase[0][0];
  const palmBaseYCoordinate = hand[0]?.annotations?.palmBase[0][1];
  const yScreenPercentage = ((palmBaseYCoordinate) / videoElement.value.videoHeight);
  controlsY.value =  yScreenPercentage;
  
  // the percentage of the screen that palmBaseXCoordinate is in the controlBoundingBox
  // const boundPercentage = (palmBaseXCoordinate - controlBoundingBox.x) / controlBoundingBox.width;
  // const boundPercentageInverted = 100 - boundPercentage * 100;
  // // scroll boundPercentage in X for tickets html container
  // tickets.value.scrollLeft = boundPercentage * tickets.value.scrollWidth;
  
  // if (!isInRange(yScreenPercentage, lastPercentage - 0.005, lastPercentage + 0.005)) {
  //   lastPercentage = yScreenPercentage;
  //   scrollModal.value = lastPercentage;
  // }
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
        const timer = window.setInterval(() => {
          detectBarcodes();
          detectHand(net, video);
          // window.clearInterval(timer);
        }, 50);
      })
      .catch((error) => {
        console.log("Something went wrong!", error);
      });
  }
}
</script>

<style scoped>
.app {
  padding: 0;
}
</style>