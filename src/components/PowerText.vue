<template>
  <div>
    <div ref="settings">
      <h3>Text settings</h3>
      <input
        type="checkbox"
        v-model="showText"
      > Show text?
      <br>
      <input
        class="digits"
        v-model="digits"
        type="number"
      > digits
    </div>
    <div
      class="pwr-text-container"
      @input="inputHandler($event)"
      v-show="showText"
    >
      <p>
        <b>Test</b>: A size α=
        <input
          class="float frac01 propagate-edit statistic-alpha"
          ref="alpha"
          :value="alpha_display"
        >
        test of the null hypothesis that the effect size {{ es_type }} is
        <select
          class="text propagate-edit statistic-side"
          v-model="side"
        >
          <option
            v-for="s in sideOptions"
            :key="s.text"
            :value="s.value"
          >
            {{ s.text }}
          </option>
        </select>
        <input
          class="float propagate-edit statistic-es0"
          ref="es0"
          v-model.number="es0"
        >. This hypothesis will be rejected when there is evidence
        that the effect size is that {{ signAlt }} {{ es0 }}. The criterion for this test is
        {{ criterion_on }} = {{ this.round(criterion) }}.
      </p>
      <p>
        <b>Design</b>: The selected design has N=(<input
          class="integer positive propagate-edit statistic-n1"
          ref="n1"
          v-model.number="n1"
        >, {{ n2 }}), for a total of
        {{ ntotal }} participants.
      </p>
      <p>
        <b>Power</b>
        <ul
          class="power_list"
        >
          <li>
            This design has power of at least
            <input
              class="float frac01 propagate-edit statistic-power"
              ref="power"
              :value="power_display"
            >
            to detect effect sizes of {{ signAlt }}
            <input
              class="float propagate-edit statistic-es"
              ref="es"
              :value="es_display"
            >
          </li>
          <li>This design has at least 0.5 power to detect effect sizes of {{ signAlt }} {{ this.round(es50) }} </li>
          <li>This design has a power of at least 1-α={{ this.round(1 - alpha) }} to detect effect sizes of {{ signAlt }} {{ this.round(es1mAlpha) }}</li>
          <li
            v-if="showTypeS"
          >
            The probability of a Type S error when {{ es_type }} {{ signAlt }} {{ this.round(es) }} is {{ signNull }} {{ this.round(typeS) }}
          </li>
          <li>
            A just-significant result will lead to a {{ CIcoef_display }}% CI on  {{ es_type }} of width
            <input
              class="float propagate-edit statistic-precision2Alpha"
              ref="precision2Alpha"
              :value="precision2Alpha_display"
            >
          </li>
        </ul>
      </p>
    </div>
    <span
      ref="scratch"
      class="scratch"
    />
  </div>
</template>

<script>

export default {
  name: 'PowerText',
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
        return [0.5, 0.999]
      }
    },
    minESDiff: {
      type: Number,
      default: 0.001
    },
    initReport: {
      required: true,
      type: Object
    }
  },
  data: function () {
    return {
      sideOptions: [
        { value: 1, text: 'at most' },
        { value: -1, text: 'at least' }
      ],
      idx: 0,
      digits: 4,
      designReport: [this.initReport],
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
      criterion: this.initReport.test.criterion,
      criterion_on: this.initReport.test.criterion_on,
      n1: this.initReport.design.n1,
      n2: this.initReport.design.n2,
      ntotal: this.initReport.design.ntotal,
      showTypeS: true,
      typeS: this.initReport.curve.typeS,
      showText: true
    }
  },
  mounted () {
    // Move all settings to global settings
    this.$parent.$refs.settings.appendChild(this.$refs.settings)

    this.updateText(this.designReport)
  },
  computed: {
    minPower: function () {
      return this.alpha
    },
    esLimits: function () {
      return this.side < 0 ? [-Infinity, this.es0 - this.minESDiff] : [this.es0 + this.minESDiff, Infinity]
    },
    signNull: function () {
      return this.side > 0 ? 'at most' : 'at least'
    },
    signAlt: function () {
      return this.side > 0 ? 'greater than' : 'less than'
    },
    CIcoef_display: function () {
      const cc = (1 - 2 * this.alpha)
      return this.round(cc * 100, -2)
    },
    alpha_display: function () {
      return this.round(this.alpha)
    },
    power_display: function () {
      return this.round(this.power)
    },
    es_display: function () {
      return this.round(this.es)
    },
    es0_display: function () {
      return this.es0
    },
    precision2Alpha_display: function () {
      return this.round(this.precision2Alpha)
    },
    n1_display: function () {
      return this.n1
    }
  },
  methods: {
    round: function (v, adj = 0, fixed = true) {
      if (fixed) {
        // https://stackoverflow.com/a/20439411/1129889
        const r = (+v).toFixed(this.digits + adj).replace(/([0-9]+(\.[0-9]+[1-9])?)(\.?0+$)/, '$1')
        return r
      } else {
        const p10 = Math.pow(10, this.digits + adj)
        return Math.round(v * p10) / p10
      }
    },
    inputHandler: function ($event) {
      if (String($event.target.value).trim() === '') return

      const classList = $event.target.classList
      const obj = { src: 'text' }
      let el = null
      classList.forEach(function (element) {
        const match = element.match(/statistic-([a-z0-9]+)/i)
        if (match != null) {
          el = match[1]
        }
      })
      if (el === null) {
        return
      }
      if (classList.contains('propagate-edit')) {
        if (classList.contains('statistic-n1')) {
          obj.n1 = this.n1
        }

        // All floats below
        const v = parseFloat($event.target.value)
        if (isNaN(v)) return

        if (classList.contains('statistic-alpha')) {
          obj.test = { alpha: v }
        }
        if (classList.contains('statistic-es0')) {
          obj.test = { es0: v }
        }
        if (classList.contains('statistic-power')) {
          obj.curve = { power: v }
        }
        if (classList.contains('statistic-es')) {
          obj.curve = { es: v }
        }
        if (classList.contains('statistic-precision2Alpha')) {
          obj.precision_2alpha = v
        }
      }
      this.$emit('request-power-change', obj)
    },
    inputResize: function (which) {
      this.$refs.scratch.innerHTML = this[`${which}_display`]
      const w = this.$refs.scratch.getBoundingClientRect().width
      this.$refs.scratch.innerHTML = ''
      this.$refs[which].style.width = `${w}px`
    },
    updateText: function (designReport, src, idx = 0) {
      this.designReport = designReport
      this.idx = idx
      this.updateLocal()
    },
    updateLocal: function () {
      const idx = this.idx
      const designReport = this.designReport[idx]
      this.es0 = designReport.test.es0
      this.alpha = designReport.test.alpha
      this.side = designReport.test.side
      this.es = designReport.curve.point.es
      this.power = designReport.curve.point.power
      this.es1mAlpha = designReport.curve.es1mAlpha
      this.es50 = designReport.curve.es50
      this.precision2Alpha = designReport.precision_2alpha
      this.criterion = designReport.test.criterion
      this.es_type = designReport.test.es_type
      this.criterion_on = designReport.test.criterion_on
      this.ulimit = Object.is(designReport.curve.powerLimit, undefined) ? 1 : designReport.curve.powerLimit
      this.hasLimit = !Object.is(designReport.curve.powerLimit, undefined)
      this.n1 = designReport.design.n1
      this.n2 = designReport.design.n2
      this.ntotal = designReport.design.ntotal
      this.typeS = designReport.curve.typeS
    }
  },
  watch: {
    precision2Alpha: {
      handler: function () {
        this.inputResize('precision2Alpha')
      },
      immediate: false
    },
    alpha: {
      handler: function () {
        this.inputResize('alpha')
      },
      immediate: false
    },
    n1: {
      handler: function () {
        this.inputResize('n1')
      },
      immediate: false
    },
    power: {
      handler: function () {
        this.inputResize('power')
      },
      immediate: false
    },
    es: {
      handler: function () {
        this.inputResize('es')
      },
      immediate: false
    },
    es0: {
      handler: function () {
        this.inputResize('es0')
      },
      immediate: false
    },
    side: {
      handler: function () {
        this.$emit('request-power-change', { test: { side: this.side } })
      },
      immediate: false
    }
  }
}
</script>

<style lang="scss" scoped>

input.propagate-edit {
  border: 1px solid rgba(0,0,0,0);
  background: #eee;
  &:focus, &:hover {
    border: 1px solid gray;
  }

}

select.propagate-edit {
  border: 1px solid rgba(0,0,0,0);
  background: #eee;
  font-style: italic;
  &:focus, &:hover {
    border: 1px solid gray;
  }
}

span.scratch {
  visibility: hidden;
}

ul.power_list {
  list-style: initial;
    margin: initial;
    padding: 0 0 0 40px;
}

ul.power_list > li {
  display: list-item;
}

p {
  display: block;
  margin-top: 1em;
  margin-bottom: 1em;
  margin-left: 0;
  margin-right: 0;
}

input.digits {
  width: 4em;
}

</style>
