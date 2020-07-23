<template>
    <svg
            ref="lineChart"
            class="line-chart"
            :class="color"
            :viewBox="viewBox"
            @mousemove="onSVGMouseOverStateChanged"
    >
        <g class="line-chart__gridy">
            <line
                    v-for="(point, index) in gridYPoints"
                    :key="index"
                    :x1="point.x1"
                    :x2="point.x2"
                    :y1="point.y1"
                    :y2="point.y2"
            ></line>
        </g>

        <g class="line-chart__surface">
            <path :d="surfacePoints"></path>
        </g>

        <g class="line-chart__axisx-label">
            <text
                    v-for="(point, index) in gridXPoints"
                    :key="index"
                    :x="point.x1"
                    :y="point.y2 + 16"
                    text-anchor="middle"
            >
                {{ point.value }}
            </text>
        </g>
        <g class="line-chart__axisy-label">
            <text
                    v-for="(point, index) in gridYPoints"
                    :key="index"
                    :x="point.x1 - 8"
                    :y="point.y1"
                    text-anchor="end"
            >
                {{ point.value }}
            </text>
        </g>

        <g v-if="currentData" class="line-chart__support-grid">
            <line
                    :x1="currentData.x"
                    :x2="currentData.x"
                    :y1="currentData.y - tooltipTextOffsetY"
                    :y2="box.bottom"
            ></line>
        </g>

        <g class="line-chart__line">
            <polyline :points="linePoints" />
        </g>

        <g class="line-chart__points">
            <circle
                    v-for="(point, index) in points"
                    :key="index"
                    :class="{ current: isCurrentData(point) }"
                    :cx="point.x"
                    :cy="point.y"
                    :r="pointRadius"
            ></circle>
        </g>

        <g v-if="currentData" class="line-chart__tooltip">
            <rect
                    :x="tooltipPosition.rectX"
                    :y="tooltipPosition.rectY"
                    :width="tooltipPosition.rectWidth"
                    :height="tooltipPosition.rectHeight"
                    rx="4"
                    ry="4"
            />
            <text
                    :x="tooltipPosition.textX"
                    :y="tooltipPosition.textY"
                    text-anchor="middle"
                    dominant-baseline="central"
            >
                {{ tooltipLabel }}
            </text>
        </g>
    </svg>
</template>

<script>
    const DAY = ["日", "月", "火", "水", "木", "金", "土"]
    export default {
        name: 'LineChart',
        props: {
            color: {
                type: String,
                default: "green"
            },
            graphDataList: {
                type: Array,
                default: () => []
            }
        },
        data: function() {
            return {
                width: 0,
                height: 200,
                pointRadius: 6,
                currentData: null,
                marginWidth: 36,
                marginHeight: 30,
                gridYCount: 5,
                tooltipRectWidth: 84,
                tooltipRectHeight: 20,
                tooltipTextOffsetY: 22
            }
        },
        watch: {
            graphDataList: function(newValue) {
                if (newValue && newValue.length === 0) {
                    this.currentData = null
                }
            }
        },
        mounted: function() {
            this.width = this.$refs.lineChart.clientWidth
        },
        computed: {
            viewBox: function() {
                return `0 0 ${this.width} ${this.height}`
            },
            box: function() {
                return {
                    top: this.marginHeight + this.pointRadius,
                    right: this.width - (this.marginWidth + this.pointRadius),
                    bottom: this.height - (this.marginHeight + this.pointRadius),
                    left: this.marginWidth + this.pointRadius
                }
            },
            points: function() {
                if (this.width < 0 || this.graphDataList.length === 0) {
                    return []
                }
                const distance =
                    (this.box.right - this.box.left) / (this.graphDataList.length - 1)
                const maxValue = Math.max(
                    ...this.graphDataList.map(item => item.docCount)
                )

                return this.graphDataList.map((item, index) => {
                    return {
                        x: distance * index + this.pointRadius + this.marginWidth,
                        y: this.map(
                            maxValue - item.docCount,
                            maxValue,
                            0,
                            this.box.bottom,
                            this.box.top
                        ),
                        graphData: item
                    }
                })
            },
            linePoints: function() {
                return this.points
                    .map(item => {
                        return `${item.x},${item.y}`
                    })
                    .join(",")
            },
            surfacePoints: function() {
                let d = `M${this.box.left},${this.box.bottom}`

                d += this.points
                    .map(item => {
                        return `L${item.x},${item.y}`
                    })
                    .join(" ")

                d += `L${this.box.right},${this.box.bottom} Z`

                return d
            },
            gridXPoints: function() {
                return this.points
                    .filter((_item, index) => {
                        if (index === 0 || index === this.graphDataList.length - 1) {
                            return true
                        } else {
                            return (
                                index === Math.floor(this.graphDataList.length / 3) ||
                                index === Math.floor((this.graphDataList.length / 3) * 2)
                            )
                        }
                    })
                    .map(item => {
                        const date = new Date(item.graphData.key)
                        return {
                            x1: item.x,
                            x2: item.x,
                            y1: this.box.top,
                            y2: this.box.bottom,
                            value: `${date.getMonth() + 1}月${date.getDate()}`
                        }
                    })
            },
            gridYPoints: function() {
                if (this.graphDataList.length === 0) {
                    return []
                }
                let count = this.gridYCount
                let maxValue = Math.max(...this.graphDataList.map(item => item.docCount))
                let step = 0

                if (maxValue === 0) {
                    count = this.gridYCount + 1
                    maxValue = this.gridYCount
                    step = 1
                } else if (maxValue < count) {
                    count = maxValue + 1
                    step = 1
                } else {
                    step = maxValue / count
                }

                return Array(count)
                    .fill()
                    .map((_, index) => {
                        const y = Math.floor(
                            this.map(
                                index * step,
                                0,
                                step * (count - 1),
                                this.box.top,
                                this.box.bottom
                            )
                        )
                        const value = Math.floor(
                            this.map(index * step, 0, step * (count - 1), maxValue, 0)
                        )
                        return {
                            x1: this.box.left,
                            x2: this.box.right,
                            y1: y,
                            y2: y,
                            value
                        }
                    })
            },
            tooltipLabel: function() {
                if (!this.currentData) {
                    return ""
                }
                const graphData = this.currentData.graphData
                const date = new Date(graphData.key)
                return `${graphData.docCount}件 ${date.getMonth() +
                1}月${date.getDate()}(${DAY[date.getDay()]})`
            },
            tooltipPosition: function() {
                if (!this.currentData) {
                    return {
                        rectX: 0,
                        rectY: 0,
                        rectWidth: 0,
                        rectHeight: 0,
                        textX: 0,
                        textY: 0
                    }
                }
                return {
                    rectX: this.currentData.x - this.tooltipRectWidth / 2,
                    rectY:
                        this.currentData.y -
                        this.tooltipRectHeight / 2 -
                        this.tooltipTextOffsetY,
                    rectWidth: this.tooltipRectWidth,
                    rectHeight: this.tooltipRectHeight,
                    textX: this.currentData.x,
                    textY: this.currentData.y - this.tooltipTextOffsetY
                }
            }
        },
        methods: {
            map: function(value, beforeMin, beforeMax, afterMin, afterMax) {
                if (beforeMin === beforeMax) {
                    if (value === beforeMin) {
                        return afterMin
                    } else {
                        return afterMax
                    }
                }
                return (
                    afterMin +
                    (afterMax - afterMin) * ((value - beforeMin) / (beforeMax - beforeMin))
                )
            },
            onSVGMouseOverStateChanged: function(e) {
                this.currentData = this.getNearestValue(e.offsetX)
            },
            getNearestValue: function(value) {
                const diff = []
                let index = 0
                this.points
                    .map(item => item.x)
                    .forEach((val, i) => {
                        diff[i] = Math.abs(value - val)
                        index = diff[index] < diff[i] ? index : i
                    })
                return this.points[index]
            },
            isCurrentData: function(point) {
                return this.currentData && point.x === this.currentData.x
            }
        }
    }
</script>

<style scoped>
    .line-chart {
        width: 100%;
    }
    .line-chart__surface path {
        fill: #999;
        opacity: 0.6;
    }
    .line-chart__line polyline {
        fill: none;
        stroke: #000;
        stroke-width: 2;
    }
    .line-chart__gridx line,
    .line-chart__gridy line {
        fill: none;
        stroke: #ddd;
        stroke-width: 1;
    }
    .line-chart__support-grid line {
        fill: none;
        stroke: #666;
        stroke-width: 1;
        stroke-dasharray: 2;
    }
    .line-chart__points circle {
        fill: none;
    }
    .line-chart__points circle.current {
        fill: #000;
        stroke: white;
        stroke-width: 2;
    }
    .line-chart__tooltip rect {
        fill: white;
        stroke: #ddd;
        stroke-width: 1px;
    }
    .line-chart.green .line-chart__surface path {
        fill: green;
    }
    .line-chart.green .line-chart__line polyline {
        stroke: green;
    }
    .line-chart.green .line-chart__points circle.current {
        fill: green;
    }
</style>
