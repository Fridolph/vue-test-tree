<template>
  <div class="home-container">
    <header>header</header>
    <Row class="main">
      <i-col span="8">
        <div class="detail-info" />
      </i-col>
      <i-col span="16">
        <TreeContainer />
      </i-col>
      <i-col span="24">
        <div class="other-module" />
      </i-col>
    </Row>
  </div>
</template>

<script>
import TreeContainer from "./TreeContainer";

export default {
  name: "home",
  components: {
    TreeContainer
  }
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
.home-container {
  width: 100%;
  min-height: 100vh;
}
.home-container header {
  position: fixed;
  z-index: 1000;
  background: pink;
  height: 64px;
  width: 100%;
}

.home-container .main {
  width: 100%;
  padding-top: 64px;
  overflow: hidden;
}

.detail-info {
  width: 100%;
  height: 740px;
  background: skyblue;
}

.other-module {
  background: khaki;
  height: 600px;
  width: 100%;
}
</style>
