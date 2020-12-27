<template>
  <div>
    <div
      class="pwr-rcode-container"
    >
      <div ref="settings">
        <h3>R code settings</h3>
        <input
          type="checkbox"
          v-model="showCode"
        > Show R code?
        <br>
      </div>
      <pre
        v-if="showCode"
      >
        # R code
        alpha = {{ alpha }}
        n1 = {{ n1 }}
        n2 = {{ n2 }}
        df = n1 + n2 - 2
        eff_n = n1 * n2 / (n1 + n2)
        es0 = {{ es0 }}
        es_power = {{ es }}
        es50 = {{ es50 }}
        es1mAlpha = {{ es1mAlpha }}

        criterion = qt(alpha, df, es0 * sqrt(eff_n), lower.tail = {{ lowerTail.alpha }})

        power = pt(criterion, df, es_power * sqrt(eff_n), lower.tail = {{ lowerTail.alpha }})

        # Should be about 0.5
        pt(criterion, df, es50 * sqrt(eff_n), lower.tail = {{ lowerTail.alpha }})

        # Should be about {{ 1 - alpha }}
        pt(criterion, df, es1mAlpha * sqrt(eff_n), lower.tail = {{ lowerTail.alpha }})

        # Type S error probability
        criterion_typeS = qt(alpha, df, es0 * sqrt(eff_n), lower.tail = {{ lowerTail.typeS }})
        typeS = pt(criterion_typeS, df, es_power * sqrt(eff_n), lower.tail = {{ lowerTail.typeS }})

        # Two-tailed power
        power_twotailed = typeS + power
        <div v-if="es0===0">
        # Confirm two-tailed power with {pwr}, if null es is 0
        pwr::pwr.t2n.test(n1 = n1, n2 = n2, d = es_power, sig.level = 2*alpha, alternative = "two.sided")
        </div>
      </pre>
    </div>
  </div>
</template>

<script>

export default {
  name: 'PowerRcode',
  props: {
    initReport: {
      required: true,
      type: Object
    }
  },
  data: function () {
    return {
      idx: 0,
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
      typeS: this.initReport.curve.typeS,
      showCode: false
    }
  },
  mounted () {
    // Move all settings to global settings
    this.$parent.$refs.settings.appendChild(this.$refs.settings)

    this.updateRcode(this.designReport)
  },
  computed: {
    lowerTail: function () {
      return {
        alpha: this.side > 0 ? 'FALSE' : 'TRUE',
        typeS: this.side > 0 ? 'TRUE' : 'FALSE'
      }
    },
    CIcoef: function () {
      return 1 - 2 * this.alpha
    }
  },
  methods: {
    updateRcode: function (designReport, src, idx = 0) {
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
  }
}
</script>

<style lang="scss" scoped>
</style>
