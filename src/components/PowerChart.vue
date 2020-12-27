<template>
  <div>
    <div
      ref="settings"
    >
      <h3>Chart settings</h3>
      <input
        class="digits"
        v-model="digits"
        type="number"
      > digits
      <br>
      <input
        type="checkbox"
        :value="showTypeS"
        @click="showTypeS = !showTypeS"
      > Type S, two-tailed
      <br>
      <input
        type="checkbox"
        :value="lockCurve"
        @click="lockCurve = !lockCurve"
      > Lock curve w/ red handle
      <br>
      <button
        class="handle_button"
        @click="resetHandle"
      >
        Reset handle
      </button>
      <button
        class="axes_button"
        @click="dragEnd"
      >
        Reset axes
      </button>
    </div>
    <div
      id="container"
      class="svg-container"
    >
      <svg
        :id="svgID"
      />
      <transition name="fade">
        <div
          ref="alpha_alert"
          class="alpha_alert chart_alert"
          v-if="handleDragging == 0"
        >
          α={{ alpha_display }}
        </div>
        <div
          ref="n_alert"
          class="n_alert chart_alert"
          v-if="(handleDragging == 2) | (handleDragging == 1 & !lockCurve)"
        >
          N<sub>total</sub>={{ designReport.design.ntotal }}
        </div>
      </transition>
    </div>
    <pre class="debug">{{ debugCurve }}
    </pre>
  </div>
</template>

<script>

import * as d3 from 'd3'
import { isElement } from 'lodash'

// The two sigmoid functions help to create a smooth power function by
// assuming the power function is approximately a normal CDF and
// then focusing the points on the curve in proportion to their
// curvature. Along with d3's monotone smoothing spline,
// this will allow us to plot accurate, smooth power curves with
// even a small number of points. This makes manipulation FAST.
function sigmoid (x, m = 0, s = 1, increasing = true) {
  var x0 = increasing ? (x - m) / s : -(x - m) / s
  if (x0 <= 0) {
    return Math.exp(-(x0 * x0) / 2) / 2
  } else {
    return 1 - sigmoid(-x0)
  }
}

function sigmoidInverse (p, m = 0, s = 1, increasing = true) {
  p = increasing ? p : 1 - p
  var x0
  if (p <= 0.5) {
    x0 = -Math.sqrt(-2 * Math.log(2 * p))
  } else {
    x0 = -sigmoidInverse(1 - p)
  }
  return s * x0 + m
}

function keepLimited (x, limits) {
  if (x <= limits[0]) {
    return limits[0]
  } else if (x >= limits[1]) {
    return limits[1]
  } else {
    return x
  }
}

export default {
  name: 'PowerChart',
  props: {
    alphaLimits: {
      type: Array,
      default: function () {
        return [0.001, 0.25]
      }
    },
    powerLimits: {
      type: Array,
      default: function () {
        return [0.001, 0.999]
      }
    },
    minESDiff: {
      type: Number,
      default: 0.001
    },
    svgID: {
      type: String,
      default: `svg${Math.round(Math.random() * Math.pow(10, 10))}`
    },
    curveN: {
      type: Number,
      default: 16
    },
    initReport: {
      required: true,
      type: Object
    },
    objFunctions: {
      required: true,
      type: Function
    },
    svgWidth: {
      type: Number,
      default: 700
    },
    svgHeight: {
      type: Number,
      default: 500
    },
    svgMargin: {
      type: Object,
      default: function () {
        return { top: 20, right: 20, bottom: 50, left: 50 }
      }
    },
    transitionDuration: {
      type: Number,
      default: 750
    }
  },
  data: function () {
    return {
      designReport: this.initReport,
      alpha: this.initReport.test.alpha,
      es0: this.initReport.test.es0,
      es_type: this.initReport.test.es_type,
      ulimit: Object.is(this.initReport.curve.powerLimit, undefined) ? 1 : this.initReport.curve.powerLimit,
      hasLimit: !Object.is(this.initReport.curve.powerLimit, undefined),
      side: this.initReport.test.side,
      power: this.initReport.curve.point.power,
      es: this.initReport.curve.point.es,
      es1mAlpha: this.initReport.curve.es1mAlpha,
      es50: this.initReport.curve.es50,
      precision2Alpha: this.initReport.precision_2alpha,
      initScaleX: false,
      showTypeS: false,
      lockCurve: false,
      handleDragging: -1,
      debugCurve: null,
      digits: 4
    }
  },
  created () {
    this.scaleY = d3.scaleLinear()
      .domain([0, 1])
      .range([this.height, 0])
    var handleDrag = this.handleDrag
    this.d3drag = d3
      .drag()
      .on('start', function (e, d) { handleDrag(e, d, this) })
      .on('drag', function (e, d) { handleDrag(e, d, this) })
      .on('end', function (e, d) { handleDrag(e, d, this) })
  },
  mounted () {
    // Move all settings to global settings
    this.$parent.$refs.settings.appendChild(this.$refs.settings)

    const line = this.line
    const svg = d3
      .select(`#${this.svgID}`)
      .attr('preserveAspectRatio', 'xMinYMin meet')
      .attr('viewBox', `0 0 ${this.svgWidth} ${this.svgHeight}`)
      .classed('svg-content', true)
      .append('g')
      .attr('transform', `translate(${this.svgMargin.left}, ${this.svgMargin.top})`)

    // Prepare y axis
    svg
      .append('g')
      .attr('class', 'axis axis--y')
      .call(d3.axisLeft(this.scaleY))

    // text label for the y axis
    svg.append('text')
      .attr('transform', 'rotate(-90)')
      .attr('y', 0 - this.svgMargin.left)
      .attr('x', 0 - (this.height / 2))
      .attr('dy', '1em')
      .style('text-anchor', 'middle')
      .text('Power')

    // Prepare x axis
    svg
      .append('g')
      .attr('class', 'axis axis--x')
      .attr('transform', `translate(0, ${this.height})`)
      .call(d3.axisBottom(this.scaleX))

    svg
      .append('text')
      .attr('transform',
        `translate(${this.width / 2}, ${this.height + this.svgMargin.top + 15})`)
      .style('text-anchor', 'middle')
      .text(`Effect size (${this.es_type})`)

    // Prepare mask
    svg
      .append('defs')
      .append('clipPath')
      .attr('id', 'mask')
      .style('pointer-events', 'none')
      .append('rect')
      .attr('x', 0)
      .attr('y', -2)
      .attr('width', this.width + 10)
      .attr('height', this.height + this.svgMargin.bottom + 2)

    // Prepare region of unreliability
    svg
      .append('rect')
      .attr('y', this.scaleY.range()[1])
      .attr('height', this.scaleY.range()[0] - this.scaleY.range()[1])
      .attr('class', 'relregion')
      .attr('clip-path', 'url(#mask)')

    // Prepare power curve
    svg
      .selectAll('.lineg')
      .data(this.curve)
      .enter()
      .append('g')
      .attr('class', 'lineg')
      .append('path')
      .attr('d', function (d) { return line(d.values) })
      .attr('class', function (d) { return `pline ${d.id}` })
      .attr('clip-path', 'url(#mask)')

    // Handles
    svg
      .selectAll('.handle')
      .data([
        { x: 0, y: 0, class: 'handle handle0' },
        { x: 0, y: 0, class: 'handle handle1' },
        { x: 0, y: 0, class: 'handle handle2' }
      ])
      .enter()
      .append('circle')
      .attr('class', 'handle')

    svg
      .selectAll('.handle')
      .call(this.d3drag)

    this.updateCurve(this.designReport)
  },
  computed: {
    height: function () {
      return this.svgHeight - this.svgMargin.top - this.svgMargin.bottom
    },
    width: function () {
      return this.svgWidth - this.svgMargin.left - this.svgMargin.right
    },
    minPower: function () {
      return this.alpha
    },
    esLimits: function () {
      return this.side < 0 ? [-Infinity, this.es0 - this.minESDiff] : [this.es0 + this.minESDiff, Infinity]
    },
    alpha_display: function () {
      const p10 = Math.pow(10, this.digits)
      const num = Math.round(this.alpha * p10) / p10
      return num// .toString().padEnd(this.digits + 4, ' ')
    },
    curve: function () {
      const showTypeS = this.showTypeS
      const objFunctions = this.objFunctions
      const es975 = objFunctions([[0.975]], 'find_es')[0]
      const s = (es975 - this.es0) / 2
      const m = this.es50

      const startP = sigmoid(this.es0, m, s, this.side > 0)
      const endX = objFunctions([[this.powerLimits[1]]], 'find_es')[0]
      const endP = sigmoid(endX, m, s, this.side > 0)

      const d = (endP - startP) / (this.curveN - 1)
      var xvals = Array(this.curveN)
      xvals[0] = this.es0
      xvals[this.curveN - 1] = endX

      for (var i = 1; i < (this.curveN - 1); i++) {
        xvals[i] = sigmoidInverse(startP + d * i, m, s, this.side > 0)
      }

      var yvals = objFunctions([xvals], 'find_power')
      yvals.push(
        this.alpha,
        0.5,
        // objFunctions([[this.es]], 'find_power')[0],
        1 - this.alpha,
        0.975
      )
      xvals.push(
        this.es0,
        this.es50,
        // this.es,
        this.es1mAlpha,
        es975
      )

      // This block creates an additional point on the curve to prevent it
      // from being pulled away from the high-power side while dragging
      if (this.initScaleX) {
        const maxDisplay = this.side > 0 ? this.scaleX.domain()[1] : this.scaleX.domain()[0]
        if (this.side > 0 & (maxDisplay > endX) | this.side < 0 & (maxDisplay < endX)) {
          xvals.push(maxDisplay)
          let maxDisplayPower = objFunctions([[maxDisplay]], 'find_power')[0]
          maxDisplayPower = isNaN(maxDisplayPower) ? 1 : maxDisplayPower
          yvals.push(maxDisplayPower)
        }
      }

      var power = []
      xvals.map(function (e, i) {
        power.push({ x: e, y: yvals[i] })
      })
      power.sort(function (a, b) {
        return (a.x < b.x) ? -1 : ((a.x === b.x ? 0 : 1))
      })

      var typeS = []
      var twoSided = []
      power.map(function (e, i) {
        var tys
        var tws
        if (showTypeS) {
          tys = objFunctions([[e.x], true], 'find_power')[0]
          tws = tys + e.y
        } else {
          tys = null
          tws = null
        }
        typeS.push({ x: e.x, y: tys })
        twoSided.push({ x: e.x, y: tws })
      })

      var ulimit = []
      const hasLimit = this.hasLimit
      power.map(function (e, i) {
        var lim = Infinity
        if (hasLimit) {
          lim = objFunctions([[e.x], null, true], 'find_power')[0]
        }
        ulimit.push({ x: e.x, y: lim })
      })

      const z = [
        { id: 'typeSline', values: typeS },
        { id: 'twoSidedline', values: twoSided },
        { id: 'powerline', values: power },
        { id: 'limitline', values: ulimit }
      ]
      return z
    },
    handles: function () {
      return [
        { x: this.es0, y: this.alpha, class: 'handle handle0' },
        { x: this.es, y: this.power, class: 'handle handle1' },
        { x: this.es1mAlpha, y: 1 - this.alpha, class: 'handle handle2' }
      ]
    },
    npts: function () {
      return this.curve[0].length
    },
    line: function () {
      const scaleX = this.scaleX
      const scaleY = this.scaleY
      return d3.line()
        .x(function (d) { return scaleX(d.x) })
        .y(function (d) { return scaleY(d.y) })
        .curve(d3.curveMonotoneY)
    }
  },
  watch: {
    showTypeS: {
      handler: function () {
        this.updateCurve(this.designReport)
      }
    },
    curve: {
      handler: function () {
        this.debugOut()
        if (this.initScaleX) {
          return
        }
        this.scaleX = d3.scaleLinear()
          .range([0, this.width])

        const maxX = this.objFunctions([[this.powerLimits[1]]], 'find_es')[0]
        if (this.side > 0) {
          this.scaleX.domain([this.es0, maxX])
        } else {
          this.scaleX.domain([maxX, this.es0])
        }

        this.initScaleX = true
      },
      immediate: true
    }
  },
  methods: {
    resetHandle: function () {
      this.change({ curve: { es: this.es50, power: 0.5 } })
    },
    debugOut: function () {
      this.debugCurve = JSON.stringify({ curve: this.curve, handles: this.handles })
    },
    handleDrag: function (e, d, t) {
      const handle = d3.select(t)
      const scaleX = this.scaleX
      const scaleY = this.scaleY
      if (e.type === 'end') {
        handle.classed('active', false)
        this.dragEnd()
        return
      }

      const whichHandle = handle
        .attr('class')
        .match(/handle([012])/)[1]

      this.handleDragging = whichHandle

      if (e.type === 'start') {
        handle.raise().classed('active', true)
      }

      var x
      var y
      const y0 = scaleY.invert(e.y - e.subject.y + scaleY(e.subject.y))
      const x0 = scaleX.invert(e.x - e.subject.x + scaleX(e.subject.x))
      var powLim = 1
      switch (whichHandle) {
        case '0':
          x = e.subject.x
          y = keepLimited(y0, this.alphaLimits)
          break
        case '1':
          x = keepLimited(x0, this.esLimits)
          if (this.lockCurve) {
            y = this.objFunctions([[x], false, false], 'find_power')[0]
          } else {
            y = keepLimited(y0, [
              Math.max(this.alpha, this.powerLimits[0]),
              this.powerLimits[1]
            ])
            powLim = this.objFunctions([[x], false, true], 'find_power')[0]
            if (y >= powLim) { return }
          }
          break
        case '2':
          y = e.subject.y
          x = keepLimited(x0, this.esLimits)
          powLim = this.objFunctions([[x], false, true], 'find_power')[0]
          if (y >= powLim) { return }
          break
        default:
          y = e.subject.y
          x = e.subjecy.x
          break
      }

      handle
        .datum({ x: x, y: y, class: handle.attr('class') })
        .attr('cx', function (d) { return scaleX(d.x) })
        .attr('cy', function (d) { return scaleY(d.y) })
        .attr('class', function (d) { return d.class })

      switch (whichHandle) {
        case '0':
          this.change({ test: { es0: this.es0, alpha: y } }, t)
          break
        case '1': {
          const n1 = this.designReport.design.n1
          this.change({ curve: { es: x, power: y } }, t)
          if (this.lockCurve) {
            this.change({ n1: n1 }, t)
          }
          break
        }
        case '2':
          this.change({ es1mAlpha: x }, t)
          break
        default:
          break
      }
    },
    updateCurve: function (designReport, src, refresh = false) {
      this.designReport = designReport
      this.es0 = designReport.test.es0
      this.alpha = designReport.test.alpha
      this.side = designReport.test.side
      this.es = designReport.curve.point.es
      this.power = designReport.curve.point.power
      this.es1mAlpha = designReport.curve.es1mAlpha
      this.es50 = designReport.curve.es50
      this.precision2Alpha = designReport.precision_2alpha
      this.ulimit = Object.is(designReport.curve.powerLimit, undefined) ? 1 : designReport.curve.powerLimit
      this.hasLimit = !Object.is(designReport.curve.powerLimit, undefined)
      const scaleX = this.scaleX
      const scaleY = this.scaleY
      const line = this.line

      const svg = d3.select(`#${this.svgID}`)
      // Update region of unreliability
      const x0 = this.side > 0 ? this.es0 : this.es1mAlpha
      const x1 = scaleX(x0)
      const width = Math.abs(scaleX(this.es0) - scaleX(this.es1mAlpha))
      svg
        .select('.relregion')
        .attr('x', x1)
        .attr('width', width)

      var idx
      var dh = [...this.handles]
      if (isElement(src)) {
        idx = parseInt(d3.select(src).attr('class').match(/handle([012])/)[1])
        dh.splice(idx, 1)
      }
      // Update handles
      svg
        .selectAll('.handle')
        .filter(function () { return this !== src })
        .data(dh)
        .attr('r', 5)
        .attr('cx', function (d) { return scaleX(d.x) })
        .attr('cy', function (d) { return scaleY(d.y) })
        .attr('class', function (d) { return d.class })

      // Update power curve
      const i = svg
        .selectAll('.lineg')
        .data(this.curve)

      i.select('.pline')
        .datum(function (d) { return d })
        .attr('d', function (d) { return line(d.values) })
        .attr('class', function (d) { return `pline ${d.id}` })
        .attr('clip-path', 'url(#mask)')

      svg
        .selectAll('.twoSidedline,.typeSline')
        .style('visibility', this.showTypeS ? 'visible' : 'hidden')

      if (refresh) {
        setTimeout(this.dragEnd, 10)
        // this.dragEnd()
      }
    },
    dragEnd: function () {
      const scaleX = this.scaleX
      const scaleY = this.scaleY
      const line = this.line
      const svg = d3.select(`#${this.svgID}`).transition().duration(this.transitionDuration)

      // Reset x axis back to specified limits with transition
      // (handle might disappear!)
      const maxX = this.objFunctions([[this.powerLimits[1]]], 'find_es')[0]
      if (this.side > 0) {
        scaleX.domain([this.es0, maxX])
      } else {
        scaleX.domain([maxX, this.es0])
      }

      // Transition power curve
      svg
        .selectAll('.pline')
        .attr('d', function (d) { return line(d.values) })

      // Transition region of unreliability
      const x0 = this.side > 0 ? this.es0 : this.es1mAlpha
      const x1 = scaleX(x0)
      const width = Math.abs(scaleX(this.es0) - scaleX(this.es1mAlpha))
      d3.select('.relregion')
        .transition()
        .duration(this.transitionDuration)
        .attr('x', x1)
        .attr('width', width)

      // Transition handles
      d3.select(`#${this.svgID}`)
        .selectAll('.handle')
        .sort(function (a, b) {
          return a.class.match(/handle([012])/)[1] - b.class.match(/handle([012])/)[1]
        })
        .data(this.handles)
        .attr('class', function (d) { return d.class })
        .transition()
        .duration(this.transitionDuration)
        .attr('r', 5)
        .attr('cx', function (d) { return scaleX(d.x) })
        .attr('cy', function (d) { return scaleY(d.y) })
      // Transition x axis
      svg
        .select('.axis--x')
        .call(d3.axisBottom(scaleX))

      this.handleDragging = -1
    },
    change: function (obj, src, idx = 0) {
      obj.src = src
      obj.idx = idx
      this.$emit('request-power-change', obj)
    }
  }
}
</script>

<style lang="scss" scoped>

.chart_alert {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-4em, -50%);
  z-index: 5;
  color: rgba(0,0,0,.3);
  pointer-events: none;
  font-size: 250%;
  white-space: pre;
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 1s
}

.fade-enter, .fade-leave-to {
  opacity: 0;
}

pre.debug {
  display: none;
  font-family: monospace;
}

.svg-container {
    display: inline-block;
    position: relative;
    width: 50%;
    padding-bottom: 35%;
    vertical-align: top;
    overflow: hidden;
}
.svg-content {
    display: inline-block;
    position: absolute;
    top: 0;
    left: 0;
}

/*
::v-deep circle.active {
  stroke-width: 3;
  stroke: yellow;
}
*/

::v-deep .pline {
  fill: none;
}

::v-deep .powerline {
  stroke: darkblue;
  stroke-width: 3;
}

::v-deep .typeSline {
  stroke: red;
  stroke-width: 2;
  stroke-dasharray: 4 2;
}

::v-deep .twoSidedline {
  stroke: darkgray;
  stroke-width: 2;
  stroke-dasharray: 4 2;
}

::v-deep .limitline {
  stroke: black;
  stroke-width: 1;
  stroke-dasharray: 1 1;
}

::v-deep .relregion {
  fill: rgba(0,0,0,.1);
}

/* Style the dots by assigning a fill and stroke */
::v-deep .handle {
  stroke: #000;
  stroke-width: 1;
}

::v-deep .handle0 {
  fill: #fff;
  cursor: ns-resize;
}

::v-deep .handle1 {
  fill: red;
  cursor: move;
}

::v-deep .handle2 {
  fill: #fff;
  cursor: ew-resize;
}

input.digits {
  width: 4em;
}

</style>
