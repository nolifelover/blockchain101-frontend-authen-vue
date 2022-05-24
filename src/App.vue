<script setup>
import HelloWorld from './components/HelloWorld.vue'
import TheWelcome from './components/TheWelcome.vue'
import ButtonWalletConnect from "./components/ButtonWalletConnect.vue";
</script>

<template>
  <main>
    <ButtonWalletConnect/>
  </main>
</template>

<style>
@import './assets/base.css';

main{
  margin-top:20vh;
  text-align: center;
}
</style>
