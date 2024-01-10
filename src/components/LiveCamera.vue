<template lang="pug">
div(ref="mount")
  pre {{ formats }}
  pre {{ barcodes }}
  img(v-for="rectangle in rectangleImages" :src="rectangle")
  video(ref="videoElement" autoplay playsinline muted style="display: block; position: fixed; top: 0; left: 0; z-index: 1; width: 100vw; height: 720px;")
  //- canvas#rectangles-canvas(style="display: none; position: absolute; top: 0; left: 0; z-index: 2; width: 1280px; height: 720px;" ref="rectCanvas")
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { SobelOperatorShader } from 'three/examples/jsm/shaders/SobelOperatorShader.js';
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass.js';
import { CannyOperatorShader }  from '../shaders/CannyOperatorShader.js';

const mount = ref(null);
const rectCanvas = ref(null);
const videoElement = ref(null);
const formats = ref([]);
const barcodes = ref([]);
const rectangleImages = ref([]);
let video, scene, camera, renderer, texture, material, geometry, mesh, composer;

onMounted(() => {
  initCamera();
  initThree();
  animate();
  // check supported types
  // BarcodeDetector.getSupportedFormats().then((supportedFormats) => {
  //   supportedFormats.forEach((format) => formats.value.push(format));
  // });
});

const dilationShader = {
  uniforms: {
    "tDiffuse": { value: null },
    "radius": { value: 1 },
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform int radius;
    varying vec2 vUv;
    void main() {
      vec2 tex_offset = 1.0 / vec2(textureSize(tDiffuse, 0));
      vec4 max_color = vec4(0.0);
      for(int y = -radius; y <= radius; y++) {
        for(int x = -radius; x <= radius; x++) {
          vec4 texColor = texture(tDiffuse, vUv + vec2(x, y) * tex_offset);
          if(any(greaterThan(texColor, max_color))) {
            max_color = texColor;
          }
        }
      }
      gl_FragColor = max_color;
    }
  `,
};

const AdaptativeThresholdShader = {
name: 'AdaptativeThresholdShader',
uniforms: {
  tDiffuse: { value: null },
  uResolution: { value: new THREE.Vector2(1280, 720) },
  radius: { value: 20 },
},
vertexShader: /* glsl */`
  varying vec2 vUv;
  void main() {

    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );

  }`,
fragmentShader: `
  uniform sampler2D tDiffuse;
  uniform vec2 uResolution;
  uniform int radius;
  varying vec2 vUv;

  void main() {
    vec2 tex_offset = 1.0 / uResolution; // gets the size of one texel
    float sum = 0.0; // sum of intensities in the local neighborhood
    int count = 0; // number of pixels in the local neighborhood

    for(int y = -radius; y <= radius; y++) {
      for(int x = -radius; x <= radius; x++) {
        vec3 texColor = texture2D(tDiffuse, vUv + vec2(x, y) * tex_offset).rgb;

        // Convert to grayscale
        float gray = dot(texColor, vec3(0.299, 0.587, 0.114));

        sum += gray;
        count++;
      }
    }

    // Compute the average intensity in the local neighborhood
    float avg = sum / float(count);

    // Apply threshold
    vec3 originalColor = texture2D(tDiffuse, vUv).rgb;
    float originalGray = dot(originalColor, vec3(0.299, 0.587, 0.114));
    float binary = step(avg, originalGray);

    // Output binary color
    gl_FragColor = vec4(vec3(binary), 1.0);
  }
`
};

const BinaryShader = {
name: 'BinaryShader',
uniforms: {
  uThreshold: { value: 0.2 },
  uResolution: { value: new THREE.Vector2(1280, 720) }
},
vertexShader: /* glsl */`
  varying vec2 vUv;
  void main() {

    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );

  }`,
fragmentShader: /* glsl */`
precision mediump float;

// Input texture containing the grayscale image
uniform sampler2D uTexture;

// Resolution of the input texture
uniform vec2 uResolution;

// Threshold value
uniform float uThreshold;

void main() {
    // Normalize the pixel coordinates
    vec2 uv = gl_FragCoord.xy / uResolution;

    // Sample the texture
    float gray = texture(uTexture, uv).r;

    // Perform binary thresholding
    float binary = gray > uThreshold ? 0.0 : 1.0;
    gl_FragColor = vec4(vec3(binary), 1.0);
  }

`
};

function detectBarcodes () {
  const barcodeDetector = new BarcodeDetector({ formats: ['data_matrix'] });
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
  }).catch((error) => {
    barcodes.value = error;
    console.log(error);
    alert(error);
  }).finally(() => {
    // formats.value = 'detecting...';
  });
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
          detectBarcodes();
        }, 100);
      })
      .catch((error) => {
        console.log("Something went wrong!", error);
      });
  }
}

var cube = null;

function initThree() {
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, 1280 / 720, 0.1, 1000);
  renderer = new THREE.WebGLRenderer({preserveDrawingBuffer: true});
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.domElement.style.position = 'fixed';
  renderer.domElement.style.top = 0;
  renderer.domElement.style.left = 0;
  renderer.domElement.style.zIndex = 100;
  renderer.domElement.style.width = '1280px';
  renderer.domElement.style.height = '720px';

  // mount.value.appendChild(renderer.domElement);

  texture = new THREE.VideoTexture(video);
  material = new THREE.MeshBasicMaterial({ map: texture });
  geometry = new THREE.PlaneGeometry(1280, 720);
  mesh = new THREE.Mesh(geometry, material);

  // cube = new THREE.Mesh(new THREE.BoxGeometry(100, 100, 100), new THREE.MeshBasicMaterial({ color: 0x00ffff }));
  // scene.add(cube);

  
  composer = new EffectComposer(renderer);
  composer.addPass(new RenderPass(scene, camera));

  // Canny operator pass 
  var effectCanny = new ShaderPass(CannyOperatorShader);
  effectCanny.uniforms['resolution'].value.x = 1280;
  effectCanny.uniforms['resolution'].value.y = 720;
  effectCanny.renderToScreen = true;

  // Sobel operator pass
  var effectSobel = new ShaderPass(SobelOperatorShader);
  effectSobel.uniforms['resolution'].value.x = 1280;
  effectSobel.uniforms['resolution'].value.y = 720;
  
  
  var effectBinary = new ShaderPass(BinaryShader);
  // effectBinary.uniforms['uThreshold'].value = 0.45;
  // effectBinary.uniforms['uResolution'].value.x = 1280;
  // effectBinary.uniforms['uResolution'].value.y = 720;
  // effectBinary.renderToScreen = true;
  // effectBinary.uniforms['uTexture'].value = texture;
  
  
  var effectAdaptative = new ShaderPass(AdaptativeThresholdShader);
  composer.addPass(effectBinary);
  // composer.addPass(effectAdaptative);
  // composer.addPass(effectSobel);
  composer.addPass(effectCanny);
  
  var effectDilate = new ShaderPass(dilationShader);
  effectDilate.uniforms['radius'].value = 0.5;
  effectDilate.uniforms['tDiffuse'].value = texture;
  effectDilate.renderToScreen = true;
  // composer.addPass(effectDilate);


  scene.add(mesh);
  camera.position.z = 470;
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

function detect () {
  const squares = locateWhiteSquares();
  // cut the squares from the _canvas
  const squaresImages = squares.map(square => {
    const _canvas = document.createElement('canvas');
    // add padding to the square
    _canvas.width = square.width + 200;
    _canvas.height = square.height + 200;
    const ctx = _canvas.getContext('2d');
    ctx.drawImage(videoElement.value, square.x, square.y, square.width, square.height, 0, 0, square.width, square.height);
    return _canvas.toDataURL();
  });
  rectangleImages.value = squaresImages;
}
      
function getImageSourceFromCanvas() {
  let webglCanvas = renderer.domElement;
  let tempCanvas = rectCanvas.value;
  tempCanvas.width = webglCanvas.width;
  tempCanvas.height = webglCanvas.height;
  let ctx = tempCanvas.getContext('2d');
  ctx.drawImage(webglCanvas, 0, 0, webglCanvas.width, webglCanvas.height);
  let src = cv.imread(tempCanvas);
  return src;
}
      
function drawSquaresOnCanvas(canvasId, src, squares) {
    let canvas = rectCanvas.value;
    let ctx = canvas.getContext('2d');

  // Draw the original image onto the canvas
  cv.imshow(canvasId, src);

  // Set the properties for the square
  ctx.strokeStyle = 'green';
  ctx.lineWidth = 2;

  // Draw each square
  squares.forEach(square => {
      ctx.strokeRect(square.x, square.y, square.width, square.height);
  });
}

function findContours(src) {
    let gray = new cv.Mat();
    cv.cvtColor(src, gray, cv.COLOR_RGBA2GRAY, 0);
    let thresh = new cv.Mat();
    cv.threshold(gray, thresh, 100, 255, cv.THRESH_BINARY);
    let contours = new cv.MatVector();
    let hierarchy = new cv.Mat();
    cv.findContours(thresh, contours, hierarchy, cv.RETR_EXTERNAL, cv.CHAIN_APPROX_SIMPLE);
    gray.delete(); // Clean up
    thresh.delete();
    hierarchy.delete();
    return contours;
}

function filterContours(contours, minimumArea) {
    let filteredContours = new cv.MatVector();
    for (let i = 0; i < contours.size(); ++i) {
        let cnt = contours.get(i);
        if (cv.contourArea(cnt) > minimumArea) {
            filteredContours.push_back(cnt);
        }
    }
    return filteredContours;
}

function filterByExtent(contours) {
    let squares = [];
    for (let i = 0; i < contours.size(); ++i) {
        let cnt = contours.get(i);
        let rect = cv.boundingRect(cnt);
        let aspectRatio = rect.width / rect.height;
        if (aspectRatio > 0.8 && aspectRatio < 1.2) {
            squares.push(rect);
        }
    }
    return squares;
}

function extractCoordinates(squares) {
    return squares.map(square => {
        return { x: square.x, y: square.y, width: square.width, height: square.height };
    });
}

function filterTopThreeSquares(squares) {
    // Calculate the deviation of each square's aspect ratio from 1
    let squaresWithAspectRatio = squares.map(square => {
        let aspectRatio = square.width / square.height;
        return {
            ...square,
            aspectRatioDeviation: Math.abs(1 - aspectRatio) // Closer to 0 is more square-like
        };
    });

    // Sort the squares by their aspect ratio deviation, ascending
    squaresWithAspectRatio.sort((a, b) => a.aspectRatioDeviation - b.aspectRatioDeviation);

    // Return the top three most square-like shapes
    return squaresWithAspectRatio.slice(0, 3);
}


function locateWhiteSquares() {
  const src = getImageSourceFromCanvas();
  const contours = findContours(src);
  const filteredContours = filterContours(contours, 200);
  const squares = filterByExtent(filteredContours);
  const coordinates = extractCoordinates(squares);
  drawSquaresOnCanvas('rectangles-canvas', src, squares);
  src.delete();
  contours.delete();
  filteredContours.delete();
  let top3 = filterTopThreeSquares(coordinates);
  return top3;
}

function animate() {
  // window.setTimeout(() => {
  //   requestAnimationFrame(animate);
  //   detect();
  // }, 1000 / 100);
  // composer.render();
}
</script>

<style scoped>
div {
  width: 100%;
  height: 100vh;
}
</style>