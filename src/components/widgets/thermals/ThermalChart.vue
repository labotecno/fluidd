<template>
  <div class="chart" :style="{ 'height': height }">
    <ECharts
      ref="chart"
      :option="options"
      :setOptionOps="{ notMerge: true }"
      :initOpts="{ renderer: 'svg' }"
      :events="[
        ['legendselectchanged', handleLegendSelectChange ]
      ]">
    </ECharts>
  </div>
</template>

<script lang='ts'>
import { Vue, Component, Watch, Prop } from 'vue-property-decorator'
import { ECharts } from 'echarts'
import { getKlipperType } from '@/store/helpers'

@Component({})
export default class ThermalChart extends Vue {
  @Prop({ type: String, default: '100%' })
  height!: string;

  loading = true
  options: any = {}

  handleLegendSelectChange (e: { name: string; type: string; selected: {[index: string]: boolean } }) {
    this.$store.dispatch('charts/saveSelectedLegends', e.selected)
    if (
      this.chart &&
      !this.loading
    ) {
      this.chart.setOption({
        grid: this.grid,
        yAxis: [
          {},
          { show: this.showPowerAxis(e.selected) }
        ]
      })
    }
  }

  get chart () {
    const ref = this.$refs.chart as any
    if (ref) return ref.inst as ECharts
  }

  get chartData () {
    return this.$store.getters['charts/getChartData']
  }

  get chartSelectedLegends () {
    return this.$store.getters['charts/getSelectedLegends']
  }

  get chartRetension () {
    return this.$store.getters['charts/getChartRetention']
  }

  get grid () {
    let right = 0
    if (this.showPowerAxis(this.chartSelectedLegends)) {
      right = (this.isMobile) ? 35 : 70
    } else {
      right = (this.isMobile) ? 15 : 24
    }

    return {
      top: 30,
      left: (this.isMobile) ? 35 : 70,
      right,
      bottom: (this.isMobile) ? 55 : 50
    }
  }

  @Watch('chartData')
  onDataChange (data: any) {
    if (this.chart && !this.loading) {
      this.chart.setOption({
        dataset: {
          source: data
        }
      })
    }
  }

  mounted () {
    this.init()
    this.loading = false
  }

  beforeDestroy () {
    if (typeof window === 'undefined') return
    if (this.chart) {
      this.chart.dispose()
    }
  }

  init () {
    // Create the base options.
    this.initOptions()

    // Create the series and associated legends.
    const dataKeys = Object.keys(this.chartData[0])
    const keys = this.$store.getters['printer/getChartableSensors'] as string[]

    keys.forEach((key) => {
      let label = key
      if (key.includes(' ')) label = key.split(' ')[1]

      this.createSeries(label, key)
      if (dataKeys.includes(label + 'Target')) this.createSeries(label + 'Target', key)
      if (dataKeys.includes(label + 'Power')) this.createSeries(label + 'Power', key)
      if (dataKeys.includes(label + 'Speed')) this.createSeries(label + 'Speed', key)
    })

    // Re-apply the legend state in the store after the series has redefined the legends.
    this.$store.dispatch('charts/saveSelectedLegends', this.options.legend.selected)

    // Make sure we enable / disable the y axis.
    if (this.showPowerAxis(this.options.legend.selected)) {
      this.options.yAxis[1].show = true
    }
  }

  initOptions () {
    const darkMode = this.$store.state.config.uiSettings.theme.isDark
    const fontSize = (this.isMobile) ? 13 : 15
    const lineOpacity = 0.2
    let labelBackground = 'rgba(10,10,10,0.90)'
    let fontColor = 'rgba(255,255,255,0.25)'
    let lineColor = '#ffffff'
    if (!darkMode) {
      labelBackground = 'rgba(255,255,255,0.90)'
      fontColor = 'rgba(0,0,0,0.45)'
      lineColor = '#000000'
    }

    this.options = {
      grid: this.grid,
      legend: {
        show: false,
        top: 0,
        icon: 'circle',
        textStyle: {
          fontSize,
          color: fontColor
        },
        selected: {}
      },
      tooltip: {
        trigger: 'axis',
        confine: true,
        backgroundColor: labelBackground,
        borderColor: labelBackground,
        textStyle: {
          color: fontColor,
          fontSize
        },
        axisPointer: {
          type: 'line',
          lineStyle: {
            color: lineColor,
            opacity: lineOpacity
          },
          label: {
            color: fontColor,
            fontSize,
            backgroundColor: labelBackground
          }
        },
        position: this.tooltipPosition,
        formatter: this.tooltipFormatter
      },
      xAxis: {
        type: 'time',
        boundaryGap: false,
        max: 'dataMax',
        min: (value: any) => {
          const serverConfig = this.$store.getters['server/getConfig']
          return value.max - (serverConfig.server.temperature_store_size * 1000)
        },
        axisTick: {
          show: false
        },
        splitLine: {
          show: true,
          lineStyle: {
            color: lineColor,
            opacity: 0.05
          }
        },
        axisLabel: {
          interval: 0,
          margin: 14,
          color: fontColor,
          fontSize,
          formatter: '{H}:{mm}',
          rotate: (this.isMobile) ? 90 : 0
        },
        axisPointer: {
          label: {
            show: true,
            margin: 9,
            formatter: this.xAxisPointerFormatter
          }
        }
      },
      yAxis: [
        {
          name: (this.isMobile) ? '°C Temperature' : 'Temperature',
          nameTextStyle: {
            fontSize,
            color: fontColor,
            align: 'left'
          },
          show: true,
          type: 'value',
          position: 'left',
          splitLine: {
            show: true,
            lineStyle: {
              color: lineColor,
              opacity: 0.05
            }
          },
          minInterval: 20,
          maxInterval: 60,
          min: this.yAxisTempMin,
          max: this.yAxisTempMax,
          axisLabel: {
            interval: 0,
            margin: 14,
            color: fontColor,
            fontSize,
            formatter: (this.isMobile) ? '{value}' : '{value}°C',
            rotate: (this.isMobile) ? 90 : 0
          },
          boundaryGap: [0, '100%']
        },
        {
          name: (this.isMobile) ? 'Power %' : 'Power',
          nameTextStyle: {
            fontSize,
            color: fontColor,
            align: 'right'
          },
          show: false,
          type: 'value',
          position: 'right',
          splitLine: {
            show: false,
            lineStyle: {
              color: lineColor,
              opacity: 0.05
            }
          },
          min: 0,
          max: 1,
          axisLabel: {
            interval: 0,
            margin: 14,
            color: fontColor,
            fontSize,
            formatter: this.yAxisPowerFormatter,
            rotate: (this.isMobile) ? 90 : 0
          },
          boundaryGap: [0, '100%']
        }
      ],
      dataset: {
        source: this.chartData
      },
      dataZoom: [{
        type: 'inside',
        zoomOnMouseWheel: 'shift'
      }],
      series: []
    }
  }

  createSeries (label: string, key: string) {
    // Grab the color
    const color = Vue.$colorset.next(getKlipperType(key), key)
    const id = this.options.series.length + 1

    // Base properties
    const series: any = {
      name: label,
      id,
      type: 'line',
      yAxisIndex: 0,
      showSymbol: false,
      animation: false,
      smooth: false,
      color,
      emphasis: {
        lineStyle: {
          width: 2
        }
      },
      lineStyle: {
        color,
        type: 'solid',
        width: 2,
        opacity: 0.85
      },
      areaStyle: {
        opacity: 0.05
      },
      encode: { x: 'date', y: label }
    }

    // If this is a target, adjust its display.
    if (label.toLowerCase().endsWith('target')) {
      series.yAxisIndex = 0
      series.lineStyle.type = 'dashed'
      series.lineStyle.opacity = 0.25
      series.areaStyle.opacity = 0
    }

    // If this is a power or speed, adjust its display.
    if (
      label.toLowerCase().endsWith('power') ||
      label.toLowerCase().endsWith('speed')
    ) {
      series.yAxisIndex = 1
      series.lineStyle.type = 'dotted'
      series.lineStyle.opacity = 0.35
      series.areaStyle.opacity = 0
    }

    // Set the initial legend state (power and speed default off)
    const storedLegends = this.$store.getters['charts/getSelectedLegends']
    if (storedLegends[label] !== undefined) {
      this.options.legend.selected[label] = storedLegends[label]
    } else {
      this.options.legend.selected[label] = !(
        label.toLowerCase().endsWith('power') ||
        label.toLowerCase().endsWith('speed')
      )
    }

    // Push the series into our options object.
    this.options.series.push(series)
  }

  showPowerAxis (selected: any) {
    const filtered = Object.keys(selected)
      .filter(key => key.toLowerCase().endsWith('power') || key.toLowerCase().endsWith('speed'))
      .filter(key => selected[key] === true)

    return (filtered.length > 0)
  }

  legendToggleSelect (name: string) {
    if (this.chart) {
      this.chart.dispatchAction({
        type: 'legendToggleSelect',
        name
      })
    }
  }

  legendSelect (name: string) {
    if (this.chart) {
      this.chart.dispatchAction({
        type: 'legendSelect',
        name
      })
    }
  }

  legendUnSelect (name: string) {
    if (this.chart) {
      this.chart.dispatchAction({
        type: 'legendUnSelect',
        name
      })
    }
  }

  tooltipPosition (pos: any, params: any, el: HTMLElement, elRect: any, size: any) {
    const obj: { [index: string]: any } = { top: 0 }
    obj[['left', 'right'][+(pos[0] < size.viewSize[0] / 2)]] = 30
    return obj
  }

  tooltipFormatter (params: any) {
    let text = ''
    params
      .reverse()
      .forEach((param: any) => {
        if (
          !param.seriesName.toLowerCase().endsWith('target') &&
          !param.seriesName.toLowerCase().endsWith('power') &&
          !param.seriesName.toLowerCase().endsWith('speed') &&
          param.seriesName &&
          param.seriesName in param.value
        ) {
          text += `
            <div>
              <span style="display:inline-block;margin-right:4px;border-radius:10px;width:10px;height:10px;background-color:${param.color};"></span>
              <span style="font-size:16px;color:#666;font-weight:400;margin-left:2px">
                ${param.seriesName}:
              </span>
              <span style="float:right;margin-left:20px;font-size:16px;color:#666;font-weight:900">
                ${param.value[param.seriesName].toFixed(2)}<small>°C</small>`

          if (param.seriesName + 'Target' in param.value) {
            text += ` / ${param.value[param.seriesName + 'Target'].toFixed()}<small>°C</small>`
          }
          if (param.seriesName + 'Power' in param.value) {
            text += ` / ${(param.value[param.seriesName + 'Power'] * 100).toFixed()}<small>%</small>`
          }
          if (param.seriesName + 'Speed' in param.value) {
            text += ` / ${(param.value[param.seriesName + 'Speed'] * 100).toFixed()}<small>%</small>`
          }
          text += `</span>
            <div style="clear: both"></div>
          </div>
          <div style="clear: both"></div>`
        }
      })
    return text
  }

  get isMobile () {
    return this.$vuetify.breakpoint.mobile
  }

  xAxisPointerFormatter (params: any) {
    const d = this.$dayjs(params.value)
    return d.format('H:mm:ss')
  }

  yAxisPointerFormatter (params: any) {
    return params.value.toFixed() + '°C'
  }

  yAxisPowerFormatter (value: any) {
    return (this.isMobile) ? `${value * 100}` : `${value * 100}%`
  }

  yAxisTempMin (value: any) {
    let num1 = Math.floor(value.min / 10) * 10
    num1 = (num1 === value.min && (num1 - 10) >= 0)
      ? num1 - 10
      : num1
    return num1
  }

  yAxisTempMax (value: any) {
    let num1 = Math.ceil(value.max / 10) * 10
    num1 = (num1 === value.max)
      ? num1 + 10
      : num1
    return num1
  }
}

</script>

<style lang='scss' scoped>
  .chart {
    margin-top: 16px;
    width: 100%;
    // height: 325px;
  }
</style>
