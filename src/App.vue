<template>
  <div id="app">
    <video autoplay="true" id="video1" />
    <div style="position: relative;">
      <div
        v-for="(p, key) in points"
        :key="'p' + key"
        class="point"
        :style="{ left: p[0] + 'px', top: p[1] + 'px' }"
      ></div>
      <canvas id="canvas" :width="vidWidth" :height="vidHeight"></canvas>
    </div>
    <br />
    <input type="range" min="50" max="800" v-model="bpm" /> {{ bpm }} bpm <br />
    <input type="range" min="1" max="8" v-model="octaves" />
    {{ octaves }} octaves <br />
    <input type="range" min="50" max="800" v-model="startNote" /> freq:
    {{ startNote }} <br />
    <select v-model="scale">
      <option value="chromatic">Chromatic</option>
      <option value="pentatonic">Pentatonic</option>
      <option value="rootsFifths">Roots &amp; 5ths</option>
      <option value="roots">Roots</option>
      <option value="majorScale">Major Scale</option>
      <option value="majorTriad">Major Triad</option>
      <option value="major7">Major 7</option>
      <option value="minorScale">Minor Scale</option>
      <option value="harmonicMinorScale">Harmonic Minor Scale</option>
      <option value="minorTriad">Minor Triad</option>
      <option value="minor7">Minor 7</option>
    </select>
    {{ scale }}
    <br />
    <button @click="go">Go</button>
    <div>{{ stats }}</div>
    <div style="display: grid; grid-template-columns: 230px 230px">
      <div
        v-for="(color, key) in colors"
        :key="key"
        class="color"
        :style="{ background: `hsl(${color.h}, ${color.s}%, ${color.l}%)` }"
      >
        {{ color }}
      </div>
    </div>
  </div>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'
require('./assets/p5/p5')
require('./assets/p5/addons/p5.sound')

export default {
  name: 'App',
  components: {},
  data() {
    return {
      bpm: 200,
      scale: 'majorScale',
      vidHeight: 400,
      vidWidth: 300,
      colorsRGB: [],
      colors: [],
      points: [],
      oscs: null,
      validNotes: null,
      startNote: 220,
      octaves: 2,
      stats: {},
    }
  },
  mounted() {
    var leftVideo = document.getElementById('video1')

    var canvas = document.getElementById('canvas')
    var ctx = canvas.getContext('2d')

    if (navigator.mediaDevices.getUserMedia) {
      navigator.mediaDevices
        .getUserMedia({ video: true })
        .then(stream => {
          leftVideo.srcObject = stream
          this.vidHeight = stream.getVideoTracks()[0].getSettings().height
          this.vidWidth = stream.getVideoTracks()[0].getSettings().width

          const xUnit = Math.round(this.vidWidth / 4)
          const yUnit = Math.round(this.vidHeight / 4)
          this.points = [
            [xUnit, yUnit],
            [xUnit * 3, yUnit],
            [xUnit, yUnit * 3],
            [xUnit * 3, yUnit * 3],
          ]

          leftVideo.onplay = () => {
            function getColorAtPixel(x, y) {
              const pixel = ctx.getImageData(x, y, 1, 1)
              let r = pixel.data[0],
                g = pixel.data[1],
                b = pixel.data[2]
              return [r, g, b]
            }

            const drawVideo = () => {
              if (!leftVideo.paused && !leftVideo.ended) {
                ctx.drawImage(leftVideo, 0, 0)
                setTimeout(drawVideo, 1000 / 30)
              }
            }
            const updateColorHud = () => {
              this.colorsRGB = this.points.map(point =>
                getColorAtPixel(point[0], point[1]),
              )
              this.colors = this.colorsRGB.map(colorRGB =>
                RGBToHSL(...colorRGB),
              )
              setTimeout(updateColorHud, (1000 * 60) / this.bpm)
            }
            drawVideo()
            updateColorHud()
          }
        })
        .catch(err => {
          console.log('Something went wrong!', err)
        })
    }
  },
  watch: {
    scale() {
      this.validNotes = notesInScale(this.startNote, this.scale, this.octaves)
    },
    startNote() {
      this.validNotes = notesInScale(this.startNote, this.scale, this.octaves)
    },
    octaves() {
      this.validNotes = notesInScale(this.startNote, this.scale, this.octaves)
    },
  },
  methods: {
    go() {
      this.validNotes = notesInScale(this.startNote, this.scale, this.octaves)
      if (!this.oscs) {
        this.oscs = this.points.map(point => new p5.Oscillator())
        const types = ['sine', 'triangle', 'sawtooth', 'square']
        this.oscs.forEach((osc, i) => {
          // osc.setType(types[i]) // sine triangle sawtooth square
          osc.setType('sine')
          osc.freq(220)
          osc.amp(0.5)
          osc.start()
        })
      } else {
        this.oscs.map(osc => osc.stop())
        this.oscs = null
        return
      }
      this.updateTone()
    },
    updateTone() {
      if (!this.oscs) return
      let velocity,
        note,
        type,
        majorMinorOther = 'major'
      const typeMult = { sine: 1, square: 0.07, triangle: 0.8, sawtooth: 0.13 }

      const lightnessAvg = this.colors.reduce(
        (avg, color) => avg + color.l / this.colors.length,
        0,
      )
      this.stats.lightnessAvg = lightnessAvg
      this.bpm = lerp(this.bpm, lightnessAvg * 7, 1 / 12)

      const hueRelevantColors = this.colors.filter(
        c => c.s > 8 && c.l < 92 && c.l > 8,
      )

      const strongestColor = hueRelevantColors.reduce(
        (strongest, color) => {
          if (color.s > strongest.s) return color
          return strongest
        },
        { h: 0, s: 0, l: 0 },
      )
      if (strongestColor.h > 50 && strongestColor.h < 300)
        majorMinorOther = 'minor'
      this.stats.strongestColor = strongestColor
      // todo scale degree
      // this.startNote = strongestColor.h * 5 + 100

      let colorDiff = 0
      for (let i = 1; i < hueRelevantColors.length; i++) {
        let diff = Math.abs(hueRelevantColors[0].h - hueRelevantColors[i].h)
        if (diff > 180) diff = Math.abs(diff - 360)
        colorDiff += diff / hueRelevantColors.length
      }
      this.stats.colorDifference = colorDiff
      // todo based on this also adjust octave
      // todo center tone & range +/-
      if (colorDiff < 3) this.scale = random(['roots', 'rootsFifths'])
      if (colorDiff < 10)
        this.scale = random([majorMinorOther + 'Triad', 'roots', 'rootsFifths'])
      if (colorDiff < 20)
        this.scale = random([
          majorMinorOther + 'Triad',
          majorMinorOther + 'Scale',
          'rootsFifths',
        ])
      else if (colorDiff < 30)
        this.scale = random([
          majorMinorOther + 'Scale',
          majorMinorOther + 'Triad',
          majorMinorOther + '7',
          'pentatonic',
        ])
      else
        this.scale = random([
          majorMinorOther + 'Scale',
          majorMinorOther + '7',
          'pentatonic',
          'chromatic',
        ])

      for (let index in this.colors) {
        const { h, s, l } = this.colors[index]
        note = this.validNotes[
          Math.round((l / 100) * (this.validNotes.length - 1))
        ]
        type = 'sine'
        if (s > 20 && l < 90 && l > 10) {
          if (h < 360 * 0.33) type = 'square'
          else if (h < 360 * 0.67) type = 'triangle'
          else type = 'sawtooth'
        }
        velocity = (0.02 + s / 800) * typeMult[type]

        this.oscs[index].setType(type)
        this.oscs[index].freq(note)
        this.oscs[index].amp(velocity)
      }
      setTimeout(this.updateTone, (1000 * 60) / this.bpm)
    },
  },
}

function notesInScale(freq, scaleType = 'majorScale', octaves = 3) {
  const notes = []
  const semitones = [
    1,
    16 / 15,
    9 / 8,
    6 / 5,
    5 / 4,
    4 / 3,
    45 / 32,
    3 / 2,
    8 / 5,
    5 / 3,
    9 / 5,
    15 / 8,
  ]
  const scales = {
    chromatic: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11],
    pentatonic: [0, 2, 4, 7, 9],
    roots: [0],
    rootsFifths: [0, 7],
    majorScale: [0, 2, 4, 5, 7, 9, 11],
    majorTriad: [0, 4, 7],
    major7: [0, 4, 7, 11],
    minorScale: [0, 2, 3, 5, 7, 8, 11],
    // harmonicMinorScale: [0, 2, 3, 5, 7, 8, 11],
    minorTriad: [0, 3, 7],
    minor7: [0, 3, 7, 10],
  }
  const scale = scales[scaleType] || scales.major
  for (let octave = 0; octave <= octaves - 1; octave++) {
    let octaveBase = freq
    for (let o = 0; o < octave; o++) octaveBase *= 2
    for (let scaleDegree of scale) {
      notes.push(octaveBase * semitones[scaleDegree])
    }
  }
  return notes
}

function RGBToHSL(r, g, b) {
  // Make r, g, and b fractions of 1
  r /= 255
  g /= 255
  b /= 255

  // Find greatest and smallest channel values
  let cmin = Math.min(r, g, b),
    cmax = Math.max(r, g, b),
    delta = cmax - cmin,
    h = 0,
    s = 0,
    l = 0

  // Calculate hue
  // No difference
  if (delta == 0) h = 0
  // Red is max
  else if (cmax == r) h = ((g - b) / delta) % 6
  // Green is max
  else if (cmax == g) h = (b - r) / delta + 2
  // Blue is max
  else h = (r - g) / delta + 4

  h = Math.round(h * 60)

  // Make negative hues positive behind 360°
  if (h < 0) h += 360

  // Calculate lightness
  l = (cmax + cmin) / 2

  // Calculate saturation
  s = delta == 0 ? 0 : delta / (1 - Math.abs(2 * l - 1))

  // Multiply l and s by 100
  s = +(s * 100).toFixed(1)
  l = +(l * 100).toFixed(1)

  return { h, s, l }
}

function lerp(v0, v1, t) {
  return v0 * (1 - t) + v1 * t
}
function random(items) {
  // "~~" for a closest "int"
  return items[~~(items.length * Math.random())]
}
</script>

<style lang="scss">
video {
  display: none;
}
.color {
  width: 230px;
  height: 230px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.point {
  position: absolute;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  width: 15px;
  height: 15px;
  border: 3px solid white;
}
</style>
