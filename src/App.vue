<template>
<div id="app">
  <OrderDetector />
</div>
</template>

<script setup>
import LiveCamera from './components/LiveCamera.vue'
import OrderDetector from './components/OrderDetector.vue';
import { ref, onMounted } from 'vue';
import axios from "axios";

</script>
