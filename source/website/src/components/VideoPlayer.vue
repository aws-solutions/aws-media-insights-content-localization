<!-- 
  Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
  SPDX-License-Identifier: Apache-2.0
-->

<template>
  <b-container fluid>
    <video
      id="videoPlayer"
      ref="videoPlayer"
      width="600"
      height="300"
      data-setup="{ &quot;inactivityTimeout&quot;: 0 }"
      class="video-js vjs-fluid vjs-default-skin"
      controls
      playsinline
    >
      <source
        :src="video_url"
        type="video/mp4"
      >
      <track v-for="caption in alphabetical_caption_list" :key="caption.label" kind="captions" :src="caption.src" :srclang="caption.lang" :label="caption.label" default>
    </video>
    <canvas
      id="canvas"
      class="canvas"
    />
  </b-container>
</template>

<script>
import videojs from 'video.js'
import '@/../node_modules/video.js/dist/video-js.css'
import '@/../node_modules/videojs-markers/dist/videojs.markers.css'
import '@/../node_modules/videojs-markers/dist/videojs-markers.js'
import '@/../node_modules/videojs-hotkeys/videojs.hotkeys.min.js'

export default {
  name: 'VideoPlayer',
  props: {
    options: {
      type: Object,
      default() {
        return {};
      }
    }
  },
  data: function () {
    return {
      video_url: '',
      captions: [],
      caption_src: '',
      caption_lang: '',
      caption_label: '',
      player: null
    }
  },
  computed: {
    alphabetical_caption_list: function () {
      const alphabetical_caption_list = this.captions
      return alphabetical_caption_list.sort(function(a, b) {
        return (a.label > b.label && 1) || (a.label < b.label && -1) || 0;
      });
    }
  },
  beforeUnmount() {
    if (this.player) {
      this.player.dispose();
      this.$store.commit('updatePlayer', null)
    }
  },
  created() {
    this.video_url = this.options.sources[0].src+"#t=1.5";
    if (this.options.captions.length > 0)
    {
      this.captions = this.options.captions;
      this.caption_src = this.options.captions[0].src;
      this.caption_lang = this.options.captions[0].lang;
      this.caption_label = this.options.captions[0].label;
    }
  },
  mounted: function () {
    this.player = videojs(this.$refs.videoPlayer, {  playbackRates: [0.5, 1, 1.5, 2] });
    this.player.ready(() => {
      this.player.hotkeys({
        volumeStep: 0.1,
        seekStep: 1,
        enableVolumeScroll: false,
        enableNumbers: false,
        enableModifiersForNumbers: false
      });
      this.player.loop(true);
      this.player.markers({
        breakOverlay: {
          display: false,
        },
        markerTip: {
          display: true,
          text: function (marker) {
            return marker.text;
          }
        },
        markers: []
      });
      this.player.currentTime(0);
      let vm = this
      this.player.on('timeupdate', function() {
        let currentTime = this.currentTime()
        vm.$store.commit('updateCurrentTime', currentTime)
      });
      this.$store.commit('updatePlayer', this.player);
    });
    this.player.on('loadeddata', function() {
      // Set canvas size to videoPlayer size
      const canvas = document.getElementById('canvas');
      const video = document.getElementById('videoPlayer');
      canvas.width = video.clientWidth;
      canvas.height = video.clientHeight;
    });
    window.addEventListener('resize', function () {
      console.log("resize")
      // Update canvas size when window is resized
      const canvas = document.getElementById('canvas');
      const video = document.getElementById('videoPlayer');
      canvas.width = video.clientWidth;
      canvas.height = video.clientHeight;
    });
  },
  beforeUpdate: function () {
    this.$store.commit('updatePlayer', this.player)
  }
}
</script>

<style>
  .video-js .vjs-current-time, .vjs-no-flex .vjs-current-time {
    display: block;
  }
</style>
<style scoped>
  .canvas {
    position: absolute;
    top: 0;
    left: 30px;
    /*background-color:rgba(255,0,0,0.5);*/
    pointer-events: none;
  }
</style>
