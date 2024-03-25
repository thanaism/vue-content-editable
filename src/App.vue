<template>
  <div id="app">
    <p>エディタ1</p>
    <Akapen @blur="handleBlur"/>
    <p>エディタ2</p>
    <Akapen @blur="handleBlur"/>
  </div>

</template>

<script>
import Akapen from './components/Akapen.vue';

export default {
  name: 'App',
  components: {
    Akapen,
  },
  methods: {
    handleBlur(text) {
      console.log('handleBlur', text);
    },
  },
};
</script>