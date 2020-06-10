<template>
  <div id="app">
    <chess-board/>
  </div>
</template>

<script>
import ChessBoard from "@/components/ChessBoard";

export default {
  name: 'App',
  components: {
    ChessBoard,
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
