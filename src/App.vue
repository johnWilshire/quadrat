<template lang="pug">
v-ons-page
  v-ons-toolbar
    .center Quadrat: Cover Estimator
  v-ons-fab.cam(position='bottom left')
    v-ons-icon(icon='md-camera', @click='snapPicture')
  v-ons-speed-dial(position='bottom right', direction='up')
    v-ons-fab(@click="drawing=!drawing" :class="{drawing : drawing}")
      v-ons-icon(icon='md-edit')
    v-ons-speed-dial-item
      v-ons-icon(icon='md-delete' @click="clear")
    v-ons-speed-dial-item
      v-ons-icon(icon='md-undo' @click="undo")
  canvas(ref="myCanvas")
  v-ons-list
    v-ons-list-item 
      v-ons-col
        p  Coverage: {{ coverage }}%
      v-ons-col
        v-ons-button(@click="updateCoverage") Estimate Coverage!
    v-ons-list-item Brush Size
      v-ons-range.brush(v-model="brush" style="width: 100%;")
</template>
<script>
import Vue from 'vue'
import { fabric } from 'fabric-browseronly'
const selectionFill = 'rgba(255, 255, 255, 0.25)'
const r = 0
const g = 118
const b = 255
const drawFill = `rgb(${r},${g},${b})`
export default {
  mounted () {
    this.$refs.myCanvas.width = this.width
    this.$refs.myCanvas.height = this.height
    this.canvas = new fabric.Canvas(this.$refs.myCanvas)
    this.canvas.isDrawingMode = this.drawing
    this.canvas.freeDrawingBrush.color = drawFill

    this.initSelection()
    this.updateBackground()
    this.updateBrushWidth()
  },
  data () {
    return {
      imageSource: '/static/snek.jpg',
      canvas: null,
      selection: null,
      coverage: 0,
      drawing: false,
      brush: 50,
      debug: true,
      width: window.innerWidth,
      height: window.innerHeight / 2
    }
  },
  watch: {
    drawing () {
      this.canvas.isDrawingMode = this.drawing
    },
    brush () { this.updateBrushWidth() }
  },
  methods: {
    initSelection () {
      var margin = 5
      this.selection = new fabric.Rect({
        left: this.height / (margin * 2),
        top: this.width / (margin * 2),
        width: this.width * (margin - 1) / margin,
        height: this.height * (margin - 1) / margin,
        fill: selectionFill,
        stroke: 'white'
      })
      this.selection.lockRotation = true
      this.canvas.add(this.selection)
    },
    updateBrushWidth () {
      this.canvas.freeDrawingBrush.width = this.brush * 2 / 3
    },
    snapPicture () {
      const config = {
        quality: 50,
        destinationType: Vue.cordova.camera.DestinationType.DATA_URL
      }
      Vue.cordova.camera.getPicture((imageData) => {
        this.imageSource = 'data:image/jpeg;base64,' + imageData
        this.updateBackground()
      }, (message) => {
        console.error('Could not take photo: ' + message)
        alert('Error occured: ' + message)
      }, config)
    },
    updateBackground () {
      fabric.Image.fromURL(this.imageSource, img => {
        this.canvas.setBackgroundImage(img, this.canvas.renderAll.bind(this.canvas), {
          scaleX: this.canvas.width / img.width,
          scaleY: this.canvas.height / img.height
        })
      })
    },
    selectionPosition () {
      // for some reason we have t =o multiply this by 2
      var pos = {
        left: Math.round(this.selection.left) * 2,
        top: Math.round(this.selection.top) * 2,
        width: Math.round(this.selection.getWidth()) * 2,
        height: Math.round(this.selection.getHeight()) * 2
      }
      return pos
    },
    coverageArray (context) {
      var pos = this.selectionPosition()
      var imgData = context.getImageData(
        pos.left, pos.top,
        pos.width, pos.height
      )
      return imgData
    },
    calculateCoverage (context, name) {
      this.canvas.sendToBack(this.selection)
      var imageData = this.coverageArray(context)
      var array = imageData.data
      var total = 0
      for (var i = 0; i < array.length; i += 4) {
        var red = array[i]
        var green = array[i + 1]
        var blue = array[i + 2]
        // the pen is black so we look for pixels that are black
        // note this is pretty hacky and not robust but pixels are rarely all pure black so w.e
        if (red === r && blue === b && green === g) {
          total++
        }
      }

      if (this.debug) context.putImageData(imageData, 0, 0)

      var percent = (total / (array.length / 4) * 100).toFixed(1)
      this.coverage = percent
    },
    updateCoverage () {
      this.calculateCoverage(this.canvas.contextContainer, 'container')
    },
    undo () {
      var objs = this.canvas.getObjects()
      if (objs.length === 1) return
      this.canvas.remove(objs.pop())
      this.canvas.renderAll()
    },
    clear () {
      this.canvas.clear()
      this.updateBackground()
      this.initSelection()
    }
  }
}
</script>


<style>
.drawing,
.cam {
  background: #8bc34a;
}
.center {
  text-align: center;
}
</style>

