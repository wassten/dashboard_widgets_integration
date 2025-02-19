<template>

  <div class="widget_container fr-grid-row" :class="(loading)?'loading':''" :data-display="display" :id="widgetId">
    <LeftCol v-bind="leftColProps" v-if="leftCol || leftCol === undefined"></LeftCol>
    <div class="r_col fr-col-12" :class="{'fr-col-lg-9': leftCol}">
      <div class="chart ml-lg">
        <canvas :id="chartId"></canvas>
      </div>
    </div>
  </div>

</template>

<script>
import store from '@/store'
import Chart from 'chart.js'
import LeftCol from '@/components/LeftCol'
import { mixin } from '@/utils.js'

export default {
  name: 'MultiLineChart',
  components: {
    LeftCol
  },
  mixins: [mixin],
  data () {
    return {
      indicateur_data: undefined,
      indicateur_data2: undefined,
      labels: [],
      dataset: [],
      dataset2: [],
      widgetId: '',
      chartId: '',
      display: '',
      leftColProps: {
        localisation: '',
        currentValues: [],
        currentDate: '',
        names: [],
        evolcodes: [],
        evolvalues: [],
        isMap: false,
        date: ''
      },
      units: [],
      chart: undefined,
      loading: true,
      legendLeftMargin: 0
    }
  },
  props: {
    indicateur1: String,
    indicateur2: String,
    leftCol: {
      type: Boolean,
      default: true
    },
    lineChartConfiguration: Object
  },
  computed: {
    selectedGeoLevel () {
      return store.state.user.selectedGeoLevel
    },
    selectedGeoCode () {
      return store.state.user.selectedGeoCode
    },
    selectedGeoLabel () {
      return store.state.user.selectedGeoLabel
    },
    style () {
      return this.leftCol || this.leftCol === undefined ? 'margin-left: ' + this.legendLeftMargin + 'px' : ''
    },

  },
  methods: {

    async getData () {
      const promise1 = store.dispatch('getData', this.indicateur1).then(data => {
        this.indicateur_data = data
      })

      const promise2 = store.dispatch('getData', this.indicateur2).then(data => {
        this.indicateur_data2 = data
      })

      Promise.all([promise1, promise2]).then((values) => {
        this.loading = false
        this.createChart()
      })
    },

    updateData () {
      const self = this

      this.leftColProps.localisation = this.selectedGeoLabel

      const geolevel = this.selectedGeoLevel
      const geocode = this.selectedGeoCode

      let geoObject
      let geoObject2

      if (geolevel === 'France') {
        geoObject = this.indicateur_data.france[0]
        geoObject2 = this.indicateur_data2.france[0]
      } else {
        geoObject = this.indicateur_data[geolevel].find(obj => {
          return obj.code_level === geocode
        })
        geoObject2 = this.indicateur_data2[geolevel].find(obj => {
          return obj.code_level === geocode
        })
      }

      this.leftColProps.date = this.convertDateToHuman(geoObject.last_date)

      this.leftColProps.names.length = 0
      this.units.length = 0
      this.leftColProps.currentValues.length = 0
      this.leftColProps.evolcodes.length = 0
      this.leftColProps.evolvalues.length = 0

      this.leftColProps.names.push(this.indicateur_data.nom, this.indicateur_data2.nom)
      this.units.push(this.indicateur_data.unite, this.indicateur_data2.unite)
      this.leftColProps.currentValues.push(geoObject.last_value, geoObject2.last_value)
      this.leftColProps.currentDate = this.convertDateToHuman(geoObject.last_date)
      this.leftColProps.evolcodes.push(geoObject.evol_color, geoObject2.evol_color)
      this.leftColProps.evolvalues.push(geoObject.evol_percentage, geoObject2.evol_percentage)

      this.labels.length = 0
      this.dataset.length = 0
      this.dataset2.length = 0

      geoObject.values.forEach(function (d) {
        self.labels.push(self.convertDateToHuman(d.date))
        self.dataset.push((d.value))

        const correspondingValue = geoObject2.values.find(obj => {
          return obj.date === d.date
        })
        if (correspondingValue) {
          self.dataset2.push(correspondingValue.value)
        }
      })
    },

    updateChart () {
      this.updateData()
      this.chart.update()
    },

    createChart () {
      const self = this

      this.updateData()

      let xTickLimit
      this.display === 'big' ? xTickLimit = 6 : xTickLimit = 1

      const ctx = document.getElementById(self.chartId).getContext('2d')

      let gradientFill

      this.display === 'big' ? gradientFill = ctx.createLinearGradient(0, 0, 0, 500) : gradientFill = ctx.createLinearGradient(0, 0, 0, 250)

      gradientFill.addColorStop(0, 'rgba(218, 218, 254, 0.6)')
      gradientFill.addColorStop(0.6, 'rgba(245, 245, 255, 0)')

      let gradientFill2

      this.display === 'big' ? gradientFill2 = ctx.createLinearGradient(0, 0, 0, 350) : gradientFill2 = ctx.createLinearGradient(0, 0, 0, 225)

      gradientFill2.addColorStop(0, 'rgba(0, 124, 58, 0.6)')
      gradientFill2.addColorStop(0.6, 'rgba(0, 124, 58, 0)')

      this.chart = new Chart(ctx, this.deepMerge({
        data: {
          labels: self.labels,
          datasets: [
            {
              data: self.dataset,
              backgroundColor: gradientFill,
              borderColor: '#000091',
              type: 'line',
              pointRadius: 8,
              pointBackgroundColor: 'rgba(0, 0, 0, 0)',
              pointBorderColor: 'rgba(0, 0, 0, 0)'
            },
            {
              data: self.dataset2,
              backgroundColor: gradientFill2,
              borderColor: '#007c3a',
              type: 'line',
              pointRadius: 8,
              pointBackgroundColor: 'rgba(0, 0, 0, 0)',
              pointBorderColor: 'rgba(0, 0, 0, 0)'
            }
          ]
        },
        options: {
          maintainAspectRatio: false,
          animation: {
            easing: 'easeInOutBack'
          },
          scales: {
            xAxes: [{
              gridLines: {
                color: 'rgba(0, 0, 0, 0)'
              },
              ticks: {
                autoSkip: true,
                maxTicksLimit: xTickLimit,
                maxRotation: 0,
                minRotation: 0,
                callback: function (value) {
                  return value.toString().substring(3, 5) + '/' + value.toString().substring(8, 10)
                }
              }
            }],
            yAxes: [{
              gridLines: {
                color: '#e5e5e5',
                borderDash: [3]
              },
              ticks: {
                autoSkip: true,
                maxTicksLimit: 5,
                callback: function (value) {
                  value = parseFloat(value)
                  if (value / 1000000000 >= 1) {
                    return Intl.NumberFormat().format(value / 1000000000) + ' Mrds'
                  } else if (value / 1000000 >= 1) {
                    return Intl.NumberFormat().format(value / 1000000) + ' M'
                  } else if (value / 1000 >= 1) {
                    return Intl.NumberFormat().format(value / 1000) + ' K'
                  }
                  return Intl.NumberFormat().format(value)
                }
              },
              afterFit: function (axis) {
                self.legendLeftMargin = axis.width
              }
            }]
          },
          legend: {
            display: false
          },
          tooltips: {
            displayColors: false,
            backgroundColor: '#6b6b6b',
            callbacks: {
              label: function (tooltipItems) {
                const int = self.convertFloatToHuman(tooltipItems.value)
                return int + ' ' + self.units[tooltipItems.datasetIndex]
              },
              title: function (tooltipItems) {
                return tooltipItems[0].label
              },
              labelTextColor: function () {
                return '#eeeeee'
              }
            }
          }
        }
      }, this.lineChartConfiguration))
    }
  },

  watch: {
    selectedGeoCode: function () {
      this.updateChart()
    },
    selectedGeoLevel: function () {
      this.updateChart()
    }
  },

  created () {
    this.chartId = 'myChart' + Math.floor(Math.random() * (1000))
    this.widgetId = 'widget' + Math.floor(Math.random() * (1000))
    this.getData()
  },

  mounted () {
    document.getElementById(this.widgetId).offsetWidth > 486 ? this.display = 'big' : this.display = 'small'
  }

}
</script>

<style scoped lang="scss">

.widget_container {
  .ml-lg {
    margin-left: 0;
  }

  @media (min-width: 62em) {
    .ml-lg {
      margin-left: 3rem;
    }
  }
  @media (max-width: 62em) {
    .chart .flex {
      margin-left: 0 !important
    }
  }

  .r_col {
    align-self: center;

    .flex {
      display: flex;

      .legende_dot {
        width: 1rem;
        height: 1rem;
        border-radius: 50%;
        background-color: var(--bg500-plain);
        display: inline-block;
        margin-top: 0.25rem;

        &[data-serie="2"] {
          background-color: #007c3a;
        }
      }
    }
  }

  .chart canvas {
    max-width: 100%;
  }

}

</style>
