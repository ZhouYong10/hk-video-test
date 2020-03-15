<template>
  <div class="home">
    <fm-hk-video
      style="width:1000px;height:600px;margin-left: 600px"
      appkey="21349144"
      secret="DuyWn2H7Qb7jB9rPArth"
      ip="10.19.132.75"
    />
    <img alt="Vue logo" src="../assets/logo.png" />
    <HelloWorld msg="Welcome to Your Vue.js App" />
  </div>
</template>

<script lang="ts">
// @ is an alias to /src
import {Component, Vue} from "vue-property-decorator";
import HelloWorld from "@/components/HelloWorld.vue";
import FmHkVideo from "@/components/FmHkVideo.vue";

@Component({
  name: "Home",
  components: {
    FmHkVideo,
    HelloWorld
  }
})
export default class Home extends Vue{

};
</script>
