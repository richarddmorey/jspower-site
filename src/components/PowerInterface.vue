<template>
  <div>
    <vs-button
      @click="settingsActive=!settingsActive"
      color="primary"
      type="filled"
    >
      &#9881; Options
    </vs-button>
    <vs-sidebar
      position-right
      parent="body"
      default-index="1"
      color="primary"
      class="sidebarx"
      spacer
      v-model="settingsActive"
    >
      <div
        class="global_settings"
        ref="settings"
      >
        <div>
          <h3>Global settings</h3>
          <input
            type="checkbox"
            :value="fixES"
            @click="fixES = !fixES"
          > fix es
          <br>
          <input
            type="checkbox"
            :value="fixN2"
            @click="fixN2 = !fixN2"
          > fix n2
          <br>
          <button
            class="add_analysis_button"
            v-if="obj.length==1"
            @click="newCurve"
          >
            Add new analysis
          </button>
          <button
            class="remove_analysis_button"
            v-if="obj.length==2"
            @click="removeCurve"
          >
            Remove analysis
          </button>
          <br>
          <select
            @change="updateIdx"
            v-model="idx"
          >
            <option
              v-for="(item, index) in obj"
              :value="index"
              :key="item.id"
            >
              {{ item.id }}
            </option>
          </select>
        </div>
      </div>
    </vs-sidebar>
    <PowerChart
      :init-report="initDesign"
      :obj-functions="objFunctions"
      :method="objFunctions"
      ref="powerChart"
      @request-power-change="updatePowerObject"
    />
    <br>
    <PowerText
      :init-report="initDesign"
      ref="powerText"
      @request-power-change="updatePowerObject"
    />
    <br>
    <PowerRcode
      :init-report="initDesign"
      ref="powerRcode"
    />
  </div>
</template>

<script>
import * as pwr from 'jspower'
import PowerChart from './PowerChart'
import PowerText from './PowerText'
import PowerRcode from './PowerRcode'

import Vue from 'vue'
import Vuesax from 'vuesax'

import 'vuesax/dist/vuesax.css' // Vuesax styles
Vue.use(Vuesax, {
  // options here
})

export default {
  name: 'PowerInterface',
  components: {
    PowerChart,
    PowerText,
    PowerRcode
  },
  props: {
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
    }
  },
  data: function () {
    var t = new pwr.ttest2_pwr()
    t.test = { es0: 0, side: 1 }
    t.curve = { power: 0.8 }
    t.options.fix_es = false
    return {
      settingsActive: false,
      idx: 0,
      obj: [t],
      initDesign: t.design_report(),
      fixES: t.options.fix_es,
      fixN2: t.options.fix_n2
    }
  },
  watch: {
    fixES: {
      handler: function () {
        for (var i = 0; i < this.obj.length; i++) {
          this.obj[i].options.fix_es = this.fixES
        }
      },
      immediate: true
    },
    fixN2: {
      handler: function () {
        var designReports = []
        for (var i = 0; i < this.obj.length; i++) {
          this.obj[i].options.fix_n2 = this.fixN2
          designReports[i] = this.obj[i].design_report()
        }
        this.updatePowerObject({ })
      }
    }
  },
  methods: {
    updateIdx: function () {
      this.updatePowerObject({ src: 'switch' })
    },
    removeCurve: function () {
      const remIdx = this.idx
      if (this.obj.length === 1) return

      this.obj.splice(remIdx, 1)
      this.idx = 0
      this.updateIdx()
    },
    newCurve: function () {
      const idx = this.idx
      const fixN2 = this.obj[idx].options.fix_n2
      this.obj[idx].options.fix_n2 = false
      var t = new pwr.ttest2_pwr(
        this.obj[idx].precision_2alpha,
        this.obj[idx].nratio,
        this.obj[idx].test,
        this.obj[idx].options
      )
      t.curve = this.obj[idx].curve
      this.obj[idx].options.fix_n2 = fixN2
      t.options.fix_n2 = fixN2
      this.obj.push(t)
    },
    objFunctions: function (array, name) {
      return this.obj[this.idx][name](...array)
    },
    updatePowerObject: function (componentData) {
      const idx = this.idx

      if (componentData.test) {
        this.obj[idx].test = componentData.test
      }
      if (componentData.n1) {
        this.obj[idx].n1 = componentData.n1
      }
      if (componentData.curve) {
        this.obj[idx].curve = componentData.curve
      }
      if (componentData.es1mAlpha) {
        this.obj[idx].es1mAlpha = componentData.es1mAlpha
      }
      if (componentData.precision_2alpha) {
        this.obj[idx].precision_2alpha = componentData.precision_2alpha
      }

      const designReports = this.obj.map(function (el) {
        return el.design_report()
      })

      const refresh = ['text'].includes(componentData.src)

      this.$refs.powerChart.updateCurve(designReports[this.idx], componentData.src, refresh)
      this.$refs.powerText.updateText(designReports, componentData.src, this.idx)
      this.$refs.powerRcode.updateRcode(designReports, componentData.src, this.idx)
    }
  }
}
</script>

<style lang="scss" scoped>
h1 {
  color: black;
  font-size: 2rem;
}

button.remove_analysis_button {
  background-color: palevioletred;
}

.header-sidebar {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  width:100%;
}

.footer-sidebar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}
</style>
