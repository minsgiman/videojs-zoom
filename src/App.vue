<template>
  <div id="app">
    <button @click="zoomUp()" style="position: fixed; top:0px; left:100px;">+(zoomUp)</button>
    <button @click="zoomDown()" style="position: fixed; top:0px; left:200px;">-(zoomDown)</button>
    <p style="position: fixed; top: 0px; left: 400px;">currentZoom : {{currentZoom}}</p>
    <video id="my-player" muted class="video-js"></video>

    <div class="zoom_wrap" v-show="showZoomArea" :style="{width: zoomAreaInfo.width + 'px', height: zoomAreaInfo.height + 'px'}">
      <video id="my-player2" muted class="video-js zoom_video"></video>
      <div id="zoom_area"></div>
    </div>

  </div>
</template>

<script>
import HelloWorld from './components/HelloWorld'
import videojs from 'video.js';

let zoomMover = null;
let maskRect = null;
let player = null;

export default {
  name: 'App',
  data: function () {
      return {
          showZoomArea: false,
          currentZoom: 1,
          zoomObj: null,
          clipObj: null,
          playerSize: {width: 960, height: 540},
          zoomMoverInfo: {x: 0, y: 0, width: 0, height: 0},
          zoomAreaInfo: {width: 240, height: 135}
      }
  },
  components: {
    HelloWorld
  },
  mounted: function() {
      player = videojs('my-player', {
          controls: false,
          loop: true,
          preload: 'auto'
      });
      player.src({
          type: "video/mp4",
          src: "//vjs.zencdn.net/v/oceans.mp4"
      });
      player.on('ready', function() {
          console.log('ready');
          player.play();
      });

      const player2 = videojs('my-player2', {
          controls: false,
          loop: true,
          preload: 'auto'
      });
      player2.src({
          type: "video/mp4",
          src: "//vjs.zencdn.net/v/oceans.mp4"
      });
      player2.on('ready', function() {
          player2.play();
      });

      this.makeZoomArea('zoom_area');
  },
  methods: {
      updateVideoTransform: function() {
          const moverCenter = {
              x: this.zoomMoverInfo.x + (this.zoomMoverInfo.width / 2),
              y: this.zoomMoverInfo.y + (this.zoomMoverInfo.height / 2)
          };
          const diffWidthFromCenter = (this.zoomAreaInfo.width / 2) - moverCenter.x;
          const diffHeightFromCenter = (this.zoomAreaInfo.height / 2) - moverCenter.y;
          const calcTranslateX = diffWidthFromCenter * ((this.playerSize.width) / this.zoomAreaInfo.width);
          const calcTranslateY = diffHeightFromCenter * ((this.playerSize.height) / this.zoomAreaInfo.height);

          player.children()[0].style['transform'] = 'scale(' + this.currentZoom + ') ' +
              'translate(' + calcTranslateX + 'px, ' + calcTranslateY + 'px' + ')';
      },
      zoomUp: function() {
          if (this.currentZoom >= 5) {
              return;
          }
          this.currentZoom += 1;
          this.showZoomArea = true;
          this.updateMoverPosition();
          this.updateVideoTransform();
      },
      zoomDown: function() {
          if (this.currentZoom <= 1) {
              return;
          }
          this.currentZoom -= 1;
          this.showZoomArea = this.currentZoom === 1 ? false : true;
          this.updateMoverPosition();
          this.updateVideoTransform();
      },
      updateMoverPosition: function() {
          const centerInfo = {x: 0, y: 0};
          let x, y, width, height;

          if (this.zoomMoverInfo.width && this.zoomMoverInfo.height && this.currentZoom !== 1) {
              centerInfo.x = this.zoomMoverInfo.x + (this.zoomMoverInfo.width / 2);
              centerInfo.y = this.zoomMoverInfo.y + (this.zoomMoverInfo.height / 2);
          } else {
              centerInfo.x = this.zoomAreaInfo.width / 2;
              centerInfo.y = this.zoomAreaInfo.height / 2;
          }
          width = this.zoomAreaInfo.width / this.currentZoom;
          height = this.zoomAreaInfo.height / this.currentZoom;
          x = centerInfo.x - (width / 2);
          y = centerInfo.y - (height / 2);
          x = x < 0 ? 0 : x;
          y = y < 0 ? 0 : y;
          this.zoomMoverInfo = {x, y, width, height};

          if (zoomMover) {
              zoomMover.attr('x', this.zoomMoverInfo.x);
              zoomMover.attr('y', this.zoomMoverInfo.y);
              zoomMover.attr('width', this.zoomMoverInfo.width);
              zoomMover.attr('height', this.zoomMoverInfo.height);
          }
          if (maskRect) {
              maskRect.attr('x', this.zoomMoverInfo.x);
              maskRect.attr('y', this.zoomMoverInfo.y);
              maskRect.attr('width', this.zoomMoverInfo.width);
              maskRect.attr('height', this.zoomMoverInfo.height);
          }
      },
      makeZoomArea: function (id) {
          this.zoomObj = document.getElementById(id);
          const svg = d3.select(this.zoomObj).append('svg')
              .attr('width', '100%')
              .attr('height', '100%')
              .attr('class', 'zoom_svg');

          this.updateMoverPosition();
          this.makeZoomAreaBg(svg, this.zoomAreaInfo, this.zoomMoverInfo);
          this.makeZoomAreaMover(svg, this.zoomMoverInfo);
      },
      makeZoomAreaBg: function (svg, zoomAreaInfo, zoomMoverInfo) {
          const maskId = 'rect-mask';
          const defs = svg.append('defs');
          const mask = defs.append('mask').attr('id', maskId);

          mask.append('rect')
              .attr("x", 0).attr("y", 0).attr("width", zoomAreaInfo.width).attr("height", zoomAreaInfo.height)
              .style("fill", "white").style("opacity", 0.7);

          maskRect = mask.append('rect')
              .attr("x", zoomMoverInfo.x).attr("y", zoomMoverInfo.y).attr("height", zoomMoverInfo.height).attr("width", zoomMoverInfo.width);

          svg.append("rect")
              .attr("x", 0).attr("y", 0).attr("width", zoomAreaInfo.width).attr("height", zoomAreaInfo.height)
              .attr('fill', 'black').attr('opacity', 0.6).attr("mask", "url(#" + maskId + ")");
      },
      makeZoomAreaMover: function (svg, zoomMoverInfo) {
          const polyDragger = d3.behavior.drag()
              .on('drag', this.handlePolyDrag.bind(this))
              .on('dragend', this.handlePolyDragEnd.bind(this));
          const g = svg.append('g')
              .attr('class', 'poly_wrap');

          zoomMover = g.append('rect')
              .attr("x", zoomMoverInfo.x)
              .attr("y", zoomMoverInfo.y)
              .attr("height", zoomMoverInfo.height)
              .attr("width", zoomMoverInfo.width)
              .attr("fill", 'white')
              .attr('stroke', 'red')
              .attr('stroke-width', 0.6)
              .attr('fill-opacity', 0.1)
              .call(polyDragger);
      },
      handlePolyDrag: function () {
          this.zoomMoverInfo.x = this.zoomMoverInfo.x + d3.event.dx;
          this.zoomMoverInfo.y = this.zoomMoverInfo.y + d3.event.dy;

          if (this.zoomMoverInfo.x < 0) {
              this.zoomMoverInfo.x = 0;
          } else if ((this.zoomMoverInfo.x + this.zoomMoverInfo.width) > this.zoomAreaInfo.width) {
              this.zoomMoverInfo.x = this.zoomAreaInfo.width - this.zoomMoverInfo.width;
          }
          if (this.zoomMoverInfo.y < 0) {
              this.zoomMoverInfo.y = 0;
          } else if ((this.zoomMoverInfo.y + this.zoomMoverInfo.height) > this.zoomAreaInfo.height) {
              this.zoomMoverInfo.y = this.zoomAreaInfo.height - this.zoomMoverInfo.height;
          }
          zoomMover.attr('x', this.zoomMoverInfo.x);
          zoomMover.attr('y', this.zoomMoverInfo.y);
          maskRect.attr('x', this.zoomMoverInfo.x);
          maskRect.attr('y', this.zoomMoverInfo.y);
          this.updateVideoTransform();
      },
      handlePolyDragEnd: function () {
          console.log('handlePolyDragEnd : ' + JSON.stringify(this.zoomMoverInfo));
      }
  }
}
</script>

<style>
#app {
  position: relative;
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  width: 960px;
  height: 540px;
}
#my-player {
  position: absolute;
  overflow: hidden;
  left: 0px;
  top: 0px;
  width: 100%;
  height: 100%;
  z-index: 1;
}

.zoom_wrap {
  position: absolute;
  z-index: 10;
}
#my-player2 {
  position: absolute;
  width: 100%;
  height: 100%;
  left: 0px;
  top: 0px;
}
#zoom_area {
  position: absolute;
  width: 100%;
  height: 100%;
  left: 0px;
  top: 0px;
  cursor: pointer;
}
</style>
