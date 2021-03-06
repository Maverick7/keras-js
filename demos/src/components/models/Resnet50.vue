<template>
  <div class="demo">
    <v-progress-circular v-if="modelLoading && loadingProgress < 100" indeterminate color="primary" />
    <div class="loading-progress" v-if="modelLoading && loadingProgress < 100">Loading...{{ loadingProgress }}%</div>
    <div class="top-container" v-if="!modelLoading">
      <div class="input-container">
        <div class="input-label">Enter a valid image URL or select an image from the dropdown:</div>
        <div class="image-url">
          <v-text-field
            v-model="imageURLInput"
            label="enter image url"
            @keyup.native.enter="onImageURLInputEnter"
          ></v-text-field>
          <span>or</span>
          <v-select
            v-model="imageURLSelect"
            :items="imageURLSelectList"
            label="select image"
          ></v-select>
        </div>
      </div>
      <div class="controls">
        <mdl-switch v-model="useGPU" :disabled="modelLoading || modelRunning || !hasWebGL">Use GPU</mdl-switch>
      </div>
    </div>
    <div class="columns input-output" v-if="!modelLoading">
      <div class="column input-column">
        <div class="visualization">
          <mdl-select label="visualization" id="visualization-select"
            style="width:230px;"
            v-model="visualizationSelect"
            :options="visualizationSelectList"
          ></mdl-select>
        </div>
        <div class="canvas-container">
          <canvas id="input-canvas" width="224" height="224"></canvas>
        </div>
        <div class="loading-indicator">
          <v-progress-circular v-if="imageLoading || modelRunning" indeterminate color="primary" />
          <div class="error" v-if="imageLoadingError">Error loading URL</div>
        </div>
      </div>
      <div class="column output-column">
        <div class="output">
          <div class="inference-time">
            <span>inference time: </span>
            <span v-if="inferenceTime > 0" class="inference-time-value">{{ inferenceTime.toFixed(1) }} ms </span>
            <span v-if="inferenceTime > 0" class="inference-time-value">({{ (1000 / inferenceTime).toFixed(1) }} fps)</span>
            <span v-else>-</span>
          </div>
          <div v-for="i in [0, 1, 2, 3, 4]" :key="i"
            class="output-class" :class="{ predicted: i === 0 && outputClasses[i].probability.toFixed(2) > 0 }"
          >
            <div class="output-label">{{ outputClasses[i].name }}</div>
            <div class="output-bar"
              :style="{width: `${Math.round(100 * outputClasses[i].probability)}px`, background: `rgba(27, 188, 155, ${outputClasses[i].probability.toFixed(2)})` }"
            ></div>
            <div class="output-value">{{ Math.round(100 * outputClasses[i].probability) }}%</div>
          </div>
        </div>
      </div>
    </div>
    <div class="architecture-container" v-if="!modelLoading">
      <div v-for="(row, rowIndex) in architectureDiagramRows" :key="`row-${rowIndex}`" class="layers-row">
        <div v-for="layer in row" :key="`layer-${layer.name}`" class="layer-column">
          <div
            v-if="layer.className"
            class="layer"
            :class="{ 'has-output': finishedLayerNames.includes(layer.name) }"
            :id="layer.name"
          >
            <div class="layer-class-name">{{ layer.className }}</div>
            <div class="layer-details"> {{ layer.details }}</div>
          </div>
        </div>
      </div>
      <svg class="architecture-connections" width="100%" height="100%">
        <g>
          <path v-for="(path, pathIndex) in architectureDiagramPaths" :key="`path-${pathIndex}`" :d="path" />
        </g>
      </svg>
    </div>
  </div>
</template>

<script>
import loadImage from 'blueimp-load-image'
import ndarray from 'ndarray'
import ops from 'ndarray-ops'
import find from 'lodash/find'
import * as utils from '../../utils'
import { IMAGE_URLS } from '../../data/sample-image-urls'
import { ARCHITECTURE_DIAGRAM, ARCHITECTURE_CONNECTIONS } from '../../data/resnet50-arch'

const MODEL_FILEPATH_PROD = 'https://transcranial.github.io/keras-js-demos-data/resnet50/resnet50.bin'
const MODEL_FILEPATH_DEV = '/demos/data/resnet50/resnet50.bin'

export default {
  props: ['hasWebGL'],

  data() {
    return {
      useGPU: this.hasWebGL,
      model: new KerasJS.Model({
        filepath: process.env.NODE_ENV === 'production' ? MODEL_FILEPATH_PROD : MODEL_FILEPATH_DEV,
        gpu: this.hasWebGL,
        visualizations: ['None']
      }),
      modelLoading: true,
      modelRunning: false,
      inferenceTime: null,
      imageURLInput: '',
      imageURLSelect: null,
      imageURLSelectList: IMAGE_URLS,
      imageLoading: false,
      imageLoadingError: false,
      visualizationSelect: 'None',
      visualizationSelectList: [{ text: 'None', value: 'None' }],
      output: null,
      architectureDiagram: ARCHITECTURE_DIAGRAM,
      architectureConnections: ARCHITECTURE_CONNECTIONS,
      architectureDiagramPaths: []
    }
  },

  watch: {
    imageURLSelect(value) {
      this.imageURLInput = value
      this.loadImageToCanvas(value)
    },
    useGPU(value) {
      this.model.toggleGPU(value)
    }
  },

  computed: {
    loadingProgress() {
      return this.model.getLoadingProgress()
    },
    architectureDiagramRows() {
      let rows = []
      for (let row = 1; row < 168; row++) {
        let cols = []
        for (let col = 0; col < 2; col++) {
          cols.push(find(this.architectureDiagram, { row, col }) || {})
        }
        rows.push(cols)
      }
      return rows
    },
    finishedLayerNames() {
      // store as computed property for reactivity
      return this.model.finishedLayerNames
    },
    outputClasses() {
      if (!this.output) {
        let empty = []
        for (let i = 0; i < 5; i++) {
          empty.push({ name: '-', probability: 0 })
        }
        return empty
      }
      return utils.imagenetClassesTopK(this.output, 5)
    }
  },

  mounted() {
    this.model.ready().then(() => {
      this.modelLoading = false
      this.$nextTick(() => {
        this.drawArchitectureDiagramPaths()
      })
    })
  },

  methods: {
    drawArchitectureDiagramPaths() {
      this.architectureDiagramPaths = []
      this.architectureConnections.forEach(conn => {
        const containerElem = document.getElementsByClassName('architecture-container')[0]
        const fromElem = document.getElementById(conn.from)
        const toElem = document.getElementById(conn.to)
        const containerElemCoords = containerElem.getBoundingClientRect()
        const fromElemCoords = fromElem.getBoundingClientRect()
        const toElemCoords = toElem.getBoundingClientRect()
        const xContainer = containerElemCoords.left
        const yContainer = containerElemCoords.top
        const xFrom = fromElemCoords.left + fromElemCoords.width / 2 - xContainer
        const yFrom = fromElemCoords.top + fromElemCoords.height / 2 - yContainer
        const xTo = toElemCoords.left + toElemCoords.width / 2 - xContainer
        const yTo = toElemCoords.top + toElemCoords.height / 2 - yContainer

        let path = `M${xFrom},${yFrom} L${xTo},${yTo}`
        if (conn.corner === 'top-right') {
          path = `M${xFrom},${yFrom} L${xTo - 10},${yFrom} Q${xTo},${yFrom} ${xTo},${yFrom + 10} L${xTo},${yTo}`
        } else if (conn.corner === 'bottom-left') {
          path = `M${xFrom},${yFrom} L${xFrom},${yTo - 10} Q${xFrom},${yTo} ${xFrom + 10},${yTo} L${xTo},${yTo}`
        } else if (conn.corner === 'top-left') {
          path = `M${xFrom},${yFrom} L${xTo + 10},${yFrom} Q${xTo},${yFrom} ${xTo},${yFrom + 10} L${xTo},${yTo}`
        }

        this.architectureDiagramPaths.push(path)
      })
    },
    onImageURLInputEnter(e) {
      this.imageURLSelect = null
      this.loadImageToCanvas(e.target.value)
    },
    loadImageToCanvas(url) {
      if (!url) {
        this.clearAll()
        return
      }

      this.imageLoading = true
      loadImage(
        url,
        img => {
          if (img.type === 'error') {
            this.imageLoadingError = true
            this.imageLoading = false
          } else {
            // load image data onto input canvas
            const ctx = document.getElementById('input-canvas').getContext('2d')
            ctx.drawImage(img, 0, 0)
            this.imageLoadingError = false
            this.imageLoading = false
            this.modelRunning = true
            // model predict
            this.$nextTick(function() {
              setTimeout(() => {
                this.runModel()
              }, 200)
            })
          }
        },
        { maxWidth: 224, maxHeight: 224, cover: true, crop: true, canvas: true, crossOrigin: 'Anonymous' }
      )
    },
    runModel() {
      const ctx = document.getElementById('input-canvas').getContext('2d')
      const imageData = ctx.getImageData(0, 0, ctx.canvas.width, ctx.canvas.height)
      const { data, width, height } = imageData

      // data processing
      // see https://github.com/fchollet/keras/blob/master/keras/applications/imagenet_utils.py
      let dataTensor = ndarray(new Float32Array(data), [width, height, 4])
      let dataProcessedTensor = ndarray(new Float32Array(width * height * 3), [width, height, 3])
      ops.subseq(dataTensor.pick(null, null, 2), 103.939)
      ops.subseq(dataTensor.pick(null, null, 1), 116.779)
      ops.subseq(dataTensor.pick(null, null, 0), 123.68)
      ops.assign(dataProcessedTensor.pick(null, null, 0), dataTensor.pick(null, null, 2))
      ops.assign(dataProcessedTensor.pick(null, null, 1), dataTensor.pick(null, null, 1))
      ops.assign(dataProcessedTensor.pick(null, null, 2), dataTensor.pick(null, null, 0))

      const inputData = { input_1: dataProcessedTensor.data }
      this.model.predict(inputData).then(outputData => {
        this.inferenceTime = this.model.predictStats.forwardPass
        this.output = outputData['fc1000']
        this.modelRunning = false
        this.updateVis()
      })
    },
    updateVis() {},
    clearAll() {
      this.modelRunning = false
      this.inferenceTime = null
      this.imageURLInput = null
      this.imageURLSelect = null
      this.imageLoading = false
      this.imageLoadingError = false
      this.output = null

      this.model.finishedLayerNames = []

      const ctx = document.getElementById('input-canvas').getContext('2d')
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)
    }
  }
}
</script>

<style scoped lang="postcss">
@import '../../variables.css';

.top-container {
  margin: 10px;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: flex-start;

  & .input-container {
    width: 100%;

    & .input-label {
      font-family: var(--font-cursive);
      font-size: 16px;
      color: var(--color-lightgray);
      text-align: left;
      user-select: none;
      cursor: default;
    }

    & .image-url {
      font-family: var(--font-monospace);
      display: flex;
      flex-direction: row;
      align-items: center;
      justify-content: flex-start;
      position: relative;

      & span {
        font-family: var(--font-cursive);
        color: var(--color-lightgray);
        margin: 0 10px;
        font-size: 16px;
      }
    }
  }

  & .controls {
    width: 250px;
    margin-left: 40px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    font-family: var(--font-monospace);
  }
}

.columns.input-output {
  padding: 10px;
  margin: 0 auto;
  margin-bottom: 30px;
  background-color: whitesmoke;
  box-shadow: 3px 3px #cccccc;

  & .column {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  & .column.input-column {
    position: relative;

    & .canvas-container {
      display: inline-flex;
      justify-content: flex-end;

      & canvas {
        background: #eeeeee;
      }
    }

    & .visualization {
      font-family: var(--font-monospace);
      width: 100%;
      margin-right: 10px;
      align-self: flex-end;
    }

    & .loading-indicator {
      position: absolute;
      top: 0;
      right: 250px;
      display: flex;
      flex-direction: column;
      align-self: flex-start;

      & .mdl-spinner {
        margin: 20px;
        align-self: center;
      }

      & .error {
        color: var(--color-error);
        font-family: var(--font-monospace);
        font-size: 12px;
      }
    }
  }

  & .column.output-column {
    justify-content: flex-start;

    & .output {
      width: 370px;
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      justify-content: center;

      & .inference-time {
        align-self: center;
        font-family: var(--font-monospace);
        font-size: 14px;
        color: var(--color-lightgray);
        margin-bottom: 10px;

        & .inference-time-value {
          color: var(--color-green);
        }
      }

      & .output-class {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        padding: 6px 0;

        & .output-label {
          text-align: right;
          width: 200px;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          font-family: var(--font-monospace);
          font-size: 16px;
          color: var(--color-darkgray);
          padding: 0 6px;
          border-right: 2px solid var(--color-green-lighter);
        }

        & .output-bar {
          height: 8px;
          transition: width 0.2s ease-out;
        }

        & .output-value {
          text-align: left;
          margin-left: 5px;
          font-family: var(--font-monospace);
          font-size: 14px;
          color: var(--color-lightgray);
        }
      }

      & .output-class.predicted {
        & .output-label {
          color: var(--color-green);
          border-left-color: var(--color-green);
        }

        & .output-value {
          color: var(--color-green);
        }
      }
    }
  }
}

.architecture-container {
  min-width: 700px;
  max-width: 900px;
  margin: 0 auto;
  position: relative;

  & .layers-row {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    margin-bottom: 5px;
    position: relative;
    z-index: 1;

    & .layer-column {
      flex: 1;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 5px;

      & .layer {
        display: inline-block;
        background: whitesmoke;
        border: 2px solid whitesmoke;
        border-radius: 5px;
        padding: 2px 5px 0px;

        & .layer-class-name {
          color: var(--color-green);
          font-size: 12px;
          font-weight: bold;
        }

        & .layer-details {
          color: #999999;
          font-size: 11px;
          font-weight: bold;
        }
      }

      & .layer.has-output {
        border-color: var(--color-green);
      }
    }
  }

  & .architecture-connections {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 0;

    & path {
      stroke-width: 4px;
      stroke: #aaaaaa;
      fill: none;
    }
  }
}
</style>
