<template lang="pug">
.app-menu
  .menu-modal#menu-modal(@touchstart="" @touchmove="")
    .b(v-if="selected" style="position: fixed; width: 100%; top: calc(50% - 50px); padding: 0; z-index: 100;")
      a.selected.bigger() {{ selected }}
      //- v-progress-linear(:model-value="progress", :style="`transition: ${ progress ? 1 : 0 }s linear; background: linear-gradient(to right, #43e97b 0%, #38f9d7 100%);`" color="white")
    ul(@scroll="onScroll" ref="menu")
      li(v-for="word in words")
        a(:href="word" :class="{ '_selected': word === selected }") {{ word }}
</template>

<style>
.app-menu {
  background-color: rgba(0, 0, 0, 0.8);
  padding: 0;
  position: absolute;
  top: 0;
  left: 0;
}
.menu-modal {
  height: 100vh;
  max-height: 100vh;
  width: 100vw;
  max-width: 100vw;
  top: 0;
  left: 0;
  overflow: hidden;
}
.b {
  box-shadow: 0px 0px 50px 20px #000, 0px -500px 400px 200px rgba(0, 0, 0, 0.8), 0px -300px 400px 200px rgba(0, 0, 0, 0.5), 0px 100px 200px 100px rgba(0, 0, 0, 0.2);
  background-color: black;
}
.word:before {
  background-color: black;
  content: '';
  display: block;
  height: 100%;
  width: 100vw;
}
ul {
  height: 100vh;
  max-height: 100vh;
  padding: 50vh 0;
  margin: 0;
  overflow-y: scroll;
  filter: blur(2px);
  /* transform: translateY(-25%); */
}

a {
  font-weight: bold;
  font-size: 4rem;
  color: rgba(255,255,255,1);
  text-decoration: none;
  /* padding: 1rem; */
}

a.bigger {
  font-size: 5rem;
}

a.selected, a:hover {
  background: linear-gradient(to right, #43e97b 0%, #38f9d7 100%);
  cursor: pointer;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
</style>
  
<script setup>
import { ref, defineProps, onMounted, watch, computed } from 'vue';

const emit = defineEmits(['select']);
const cylinder = ref(null);
const props = defineProps(['items', 'scroll']);
const { scroll = null,  items = ['default'] } = props;
const words = ref(['<', ...items]);
const scrollComputed = computed(() => scroll);
const selected = ref(null);
const menu = ref(null);
const itemHeight = 90;
var deg = 0;
const progress = ref(0);
const timeout = null;


function onScroll (event) {
  const scrollPosition = event.target.scrollTop;
  const selectedIndex = Math.ceil((scrollPosition / itemHeight));
  selected.value = words.value[selectedIndex];
}

watch(props, (_props) => {
  const scrollPercentage = _props.scroll;
  menu.value.scrollTop = (scrollPercentage / 2) * menu.value.scrollHeight;
  onScroll({ target: menu.value });
});

onMounted(() => {

});
</script>