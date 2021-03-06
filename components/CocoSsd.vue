<template>
  <div id="coco-ssd">
    <div class="wrapper" ref="main"  :style="{ height: windowHeight + 'px' }">
      <video id="video" ref="video" playsinline></video>
      <canvas id="output" ref="output" v-bind:width="canvasWidth" v-bind:height="canvasHeight"/>
    </div>
  </div>
</template>

<script>
import '@tensorflow/tfjs'
import * as cocoSsd from '@tensorflow-models/coco-ssd'

export default {
  name: 'CocoSsd',
  data () {
    return {
      net: {},
      video: {},
      cameras: [],
      windowHeight: 0,
      canvasWidth: 0,
      canvasHeight: 0
    }
  },
  methods: {
    async loadCocoSsd () {
      this.net = await cocoSsd.load()
    },
    async loadVideo () {
      this.video = await this.setupCamera()
      this.video.play()
    },
    async setupCamera () {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        throw new Error(
          'Browser API navigator.mediaDevices.getUserMedia not available'
        )
      }
      const video = this.$refs.video
      this.stopExistingVideoCapture()
      const stream = await navigator.mediaDevices.getUserMedia({
        audio: false,
        video: {
          facingMode: 'user'
        }
      })
      video.srcObject = stream
      return new Promise((resolve) => {
        video.onloadedmetadata = () => {
          resolve(video)
        }
      })
    },
    async getVideoInputs () {
      if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
        window.console.log('enumerateDevices() not supported.')
        return []
      }
      const devices = await navigator.mediaDevices.enumerateDevices()
      const videoDevices = devices.filter(
        device => device.kind === 'videoinput'
      )
      return videoDevices
    },
    stopExistingVideoCapture () {
      if (this.video && this.video.srcObject) {
        this.video.srcObject.getTracks().forEach((track) => {
          track.stop()
        })
        this.video.srcObject = null
      }
    },
    doCocoSsd () {
      const canvas = this.$refs.output
      const self = this
      async function updateFrame () {
        try {
          const predictions = await self.getPredictions()
          self.drawBbox(predictions, canvas, self.video)
        } catch (e) {
          window.console.log('Retrying...')
        } finally {
          // loop
          requestAnimationFrame(updateFrame)
        }
      }
      updateFrame()
    },
    getPredictions () {
      return this.net.detect(this.video)
    },
    drawBbox (predictions, canvas, video) {
      this.canvasWidth = this.$refs.video.clientWidth
      this.canvasHeight = this.$refs.video.clientHeight
      const ctx = canvas.getContext('2d')
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)
      const font = '16px sans-serif'
      ctx.font = font
      ctx.textBaseline = 'top'
      predictions.forEach((prediction) => {
        const x = prediction.bbox[0]
        const y = prediction.bbox[1]
        const width = prediction.bbox[2]
        const height = prediction.bbox[3]
        // Draw the bounding box.
        ctx.strokeStyle = '#00FFFF'
        ctx.lineWidth = 4
        ctx.strokeRect(x, y, width, height)
        // Draw the label background.
        ctx.fillStyle = '#00FFFF'
        const textWidth = ctx.measureText(prediction.class).width
        const textHeight = parseInt(font, 10) // base 10
        ctx.fillRect(x, y, textWidth + 4, textHeight + 4)
      })

      predictions.forEach((prediction) => {
        const x = prediction.bbox[0]
        const y = prediction.bbox[1]
        // Draw the text last to ensure it's on top.
        ctx.fillStyle = '#000000'
        ctx.fillText(prediction.class, x, y)
      })
    }
  },
  async mounted () {
    this.windowHeight = window.innerHeight
    await this.loadCocoSsd()
    this.cameras = await this.getVideoInputs()
    await this.loadVideo()
    this.doCocoSsd()
  }
}
</script>

<style>
.wrapper {
  position: relative;
  text-align: center;
  margin: 0 auto;
  width: 100%,
}
#video {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  margin: auto;
}
#output {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  margin: auto;
}
</style>
