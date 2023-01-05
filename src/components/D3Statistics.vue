<template>
    <div class="statistics">
        <span>Последний промежуток: </span>
        <button @click="onButtonClick(1)">День</button>
        <button @click="onButtonClick(7)">Неделя</button>
        <button @click="onButtonClick(31)">Месяц</button>
        <button @click="onButtonClick(365)">Год</button>
        <button @click="onButtonClick(365 * 2)">2 года</button>
        <button @click="showAll">За всё время</button>
        <svg class="chart"></svg>
        <svg class="slider"></svg>
        <div class="tooltip"></div>
    </div>
</template>
<script lang='ts'>
import { defineComponent } from 'vue';
import moment from 'moment';
import * as d3 from 'd3';

import dataJson from '@/assets/aapl-bollinger.json';
import { BrushBehavior, D3BrushEvent } from 'd3';

interface StatisticsDatum {
    date: Date,
    close: number,
    lower: number,
    middle: number,
    upper: number,
}

interface ComponentData {
    chart?: D3Selection,
    barsContainer?: D3Selection,
    xAxis?: D3Selection,
    yAxis?: D3Selection,
    slider?: D3Selection,
    gBrush?: D3Selection,
    tooltip?: D3Selection,
    data: StatisticsDatum[],
    margin: {
        left: number,
        right: number,
        top: number,
        bottom: number,
    },
    width: number,
    height: number,
    sliderHeight: number,
    startDate?: Date,
    endDate?: Date,
}

type D3Selection = d3.Selection<any, any, any, any>;
enum RenderType {
    SMOOTH = 'smooth',
    INSTANT = 'instant',
}

const PARSED_DATA = dataJson.map((elem: StatisticsDatum) => ({ ...elem, date: new Date(elem.date) })) as StatisticsDatum[];
const MAXIMUM_BAR_WIDTH = 80;

export default defineComponent({
    name: 'D3Statistics',
    data(): ComponentData {
        return {
            chart: undefined,
            barsContainer: undefined,
            xAxis: undefined,
            yAxis: undefined,
            slider: undefined,
            gBrush: undefined,
            tooltip: undefined,
            data: [],
            margin: {
                left: 65,
                right: 50,
                top: 10,
                bottom: 30,
            },
            width: 1140,
            height: 410,
            sliderHeight: 60,
            startDate: undefined,
            endDate: undefined,
        };
    },
    mounted() {
        this.data = PARSED_DATA;
        this.startDate = this.data[0].date;
        this.endDate = this.data[this.data.length - 1].date;
        this.init();
    },
    methods: {
        init(): void {
            this.chart = d3.select('.chart')
                .attr('width', this.width + this.margin.left + this.margin.right)
                .attr('height', this.height + this.margin.top + this.margin.bottom);
            this.xAxis = this.chart.append('g');
            this.yAxis = this.chart.append('g');
            this.barsContainer = this.chart.append('g');
            this.slider = d3.select('.slider')
                .style('margin-left', this.margin.left)
                .attr('width', this.width)
                .attr('height', this.sliderHeight);
            this.gBrush = this.slider.append('g');
            this.initSlider();
            this.updateChart(RenderType.INSTANT);
            this.initTooltip();
        },
        updateChart(renderType: RenderType = RenderType.SMOOTH): void {
            this.renderAxises();
            this.renderBars(renderType);
        },
        renderAxises(): void {
            this.yAxis?.classed('y-axis', true)
                .call(d3.axisLeft(this.y).tickSizeOuter(0).ticks(this.height / 40))
                    .attr('transform', `translate(${this.margin.left - MAXIMUM_BAR_WIDTH / 2},0)`)
                .call(g => {
                    g.selectAll('.tick .copy').remove();
                    g.selectAll('.tick line').clone()
                        .attr('stroke-opacity', 0.15)
                        .attr('x1', 10)
                        .attr('x2', this.width + this.margin.left)
                        .classed('copy', d => d !== 1);
                });
            this.xAxis?.classed('x-axis', true)
                .attr('transform', `translate(${this.margin.left},${this.height + 3})`);
            const [firstDate, lastDate] = this.x.domain();
            const diffMoreThan5Years = d3.utcYear.offset(lastDate, -5) > firstDate;
            const diffMoreThan45Days = d3.utcMonth.offset(lastDate, -1.5) > firstDate;
            const diffMoreThan36Hours = d3.utcDay.offset(lastDate, -1.5) > firstDate;
            this.xAxis?.call(d3.axisBottom(this.x).tickSizeOuter(0).ticks(this.width / 120).tickFormat((d: Date) => {
                if (diffMoreThan5Years) return moment(d).locale('ru-RU').format('YYYY');
                if (diffMoreThan45Days) return moment(d).locale('ru-RU').format('MM.YYYY');
                if (diffMoreThan36Hours) return moment(d).locale('ru-RU').format('DD.MM');
                return moment(d).locale('ru-RU').calendar({
                    lastDay: '[Вчера,] HH:mm',
                    sameDay: '[Сегодня,] HH:mm',
                    sameElse: 'HH:mm',
                });
            }));
            this.xAxis?.select('.domain').remove();
            this.yAxis?.select('.domain').remove();
        },
        renderBars(renderType: RenderType): void {
            const duration = renderType === RenderType.SMOOTH ? 750 : 0;
            const t = d3.transition().duration(duration).ease(d3.easePoly);
            this.barsContainer?.selectAll('.bar')
                .data(this.data, (data: StatisticsDatum) => data.date)
                .join(
                    enter => enter.append('rect').classed('bar', true).attr('y', this.height).attr('x', data => this.x(data.date)),
                    update => update,
                    exit => exit.call(rect => rect.transition(t).remove().attr('y', this.height).attr('height', 0)),
                )
                .call(rect => rect.transition(t)
                    .attr('width', this.xBandwidth)
                    .attr('height', data => this.height - this.y(data.upper))
                    .attr('x', (elem) => this.x(elem.date) - this.xBandwidth / 2)
                    .attr('y', (datum) => this.y(datum.upper))
                );
            this.barsContainer?.selectAll('.bar')
                .on('mouseover', this.mouseoverBar)
                .on('mousemove', this.mousemoveBar)
                .on('mouseleave', this.mouseleaveBar);
            this.barsContainer?.transition(t).attr('transform', `translate(${this.margin.left}, 0)`);            
        },
        initSlider(): void {
            this.gBrush?.classed('slider', true)
                .call(this.brush)
                .call(this.brush.move, this.defaultBrushRange);
        },
        initTooltip(): void {
            this.tooltip = d3.select('.tooltip');
            this.tooltip.classed('tooltip', true);
        },
        mouseoverBar(): void {
            this.tooltip?.style('display', 'block');
        },
        mousemoveBar(event: MouseEvent): void {
            const data: StatisticsDatum = (event.target as any).__data__;
            this.tooltip?.html(`<div>${moment(data.date).format('DD.MM.YYYY HH:mm:ss')}<br><span class='tooltip-value'>${data.upper.toFixed()}</span> записей</div>`)
                .style('left', (d3.pointer(event, event.target)[0]) - 11 + 'px')
                .style('top', (d3.pointer(event, event.target)[1] - 25) + 'px')
        },
        mouseleaveBar(): void {
            this.tooltip?.style('display', 'none');
        },
        onButtonClick(days: number): void {
            // filter values between selected date and current date
            // todo handle brush
            const dateOffset = (24 * 60 * 60 * 1000) * days;
            const timeAgoDate = new Date();
            timeAgoDate.setTime(new Date().getTime() - dateOffset);
            this.data = PARSED_DATA.filter((elem: StatisticsDatum) => {
                return elem.date > timeAgoDate && elem.date < new Date();
            });
            this.startDate = timeAgoDate;
            this.endDate = new Date();
            this.updateChart();
        },
        showAll(): void {
            // todo handle brush
            this.data = PARSED_DATA;
            this.startDate = this.data[0].date;
            this.endDate = this.data[this.data.length - 1].date;
            this.updateChart();
        },
        onBrush({ selection }: D3BrushEvent<StatisticsDatum>): void {
            if (!selection) {
                return;
            }
            const [leftSelection, rightSelection] = selection as number[];
            const leftIndex = +this.interpolateFunc(leftSelection / this.brushX.range()[1]).toFixed();
            const rightIndex = +this.interpolateFunc(rightSelection / this.brushX.range()[1]).toFixed();
            this.data = PARSED_DATA.slice(leftIndex, rightIndex);
            this.startDate = this.data[0].date;
            this.endDate = this.data[this.data.length - 1].date;
            this.updateChart(RenderType.INSTANT);
        },
        onBrushEnd({ selection }: D3BrushEvent<StatisticsDatum>): void {
            if (selection) {
                return;
            }
            this.gBrush?.call(this.brush.move, this.defaultBrushRange);
            this.data = PARSED_DATA.slice(this.interpolateFunc(0.5), this.interpolateFunc(1));
            this.startDate = this.data[0].date;
            this.endDate = this.data[this.data.length - 1].date;
            this.updateChart();
        },
    },
    computed: {
        xBandwidth(): number {
            let prev: StatisticsDatum | undefined;
            const minimalDiff = this.data.reduce((acc, elem) => {
                if (prev) {
                    const diff = this.x(elem.date) - this.x(prev.date);
                    if (diff <= acc && diff > 0) {
                        acc = this.x(elem.date) - this.x(prev.date);
                    }
                }
                prev = elem;
                return acc;
            }, 160);
            const diffFactor = d3.median([this.data.length / PARSED_DATA.length, 1]) ?? 0.5; // coefficient for convenient bar width
            return d3.min([minimalDiff * diffFactor, MAXIMUM_BAR_WIDTH]) ?? MAXIMUM_BAR_WIDTH;
        },
        x(): d3.ScaleTime<number, number, never> {
            const xScale = d3.scaleUtc()
                .range([0, this.width])
                .domain([this.startDate ?? +new Date() - 1000, this.endDate ?? +new Date()]);
            return this.data.length > 10 ? xScale : xScale.nice();
        },
        y(): d3.ScaleLinear<number, number, never> {
            return d3.scaleLinear()
                .range([this.height, this.margin.top])
                .domain([0, d3.max(this.data, datum => datum.upper) ?? 10]);
        },
        brushX(): d3.ScaleTime<number, number, never> {
            return d3.scaleUtc()
                .range([0, this.width])
                .domain([PARSED_DATA[0].date ?? +new Date() - 1000, PARSED_DATA[PARSED_DATA.length - 1].date ?? +new Date()]);
        },
        brush(): BrushBehavior<StatisticsDatum> {
            return d3.brushX<StatisticsDatum>()
                .extent([[0, 0], [this.width, this.sliderHeight]])
                .on('brush', this.onBrush)
                .on('end', this.onBrushEnd);
        },
        defaultBrushRange(): number[] {
            return [this.width / 2, this.width];
        },
        interpolateFunc(): (t: number) => number {
            return d3.interpolate(0, PARSED_DATA.length - 1);
        },
    }
});
</script>
<style lang="scss">
.statistics {
    background-color: #f5f6f7;
    width: min-content;
    padding: 15px 20px;
}
.bar {
    cursor: pointer;
    fill: #6c8191;
    stroke-width: 2;
    stroke-opacity: 0;
    stroke: #6c8191;
    &:hover {
        fill: #4fa292;
    }
}
.tooltip {
    position: absolute;
    display: none;
    font-family: 'Roboto';
    font-size: 12px;
    font-weight: 500;
    text-align: center;
    padding: 12px;
    background-color: #e8eaed;
    border: 1px solid #aeb6be;
    border-radius: 5px;
    &-value {
        color: #4fa292;
        font-weight: bold;
    }
}
.x-axis, .y-axis {
    color: #aeb6be;
    .tick {
        color: #6c8290;
    }
}
</style>