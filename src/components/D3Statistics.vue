<template>
    <div class="statistics">
        <span>Последний промежуток: </span>
        <button @click="onButtonClick(1)">День</button>
        <button @click="onButtonClick(7)">Неделя</button>
        <button @click="onButtonClick(31)">Месяц</button>
        <button @click="onButtonClick(365)">Год</button>
        <button @click="onButtonClick(365 * 2)">2 года</button>
        <button @click="showAll">За всё время</button>
        <svg></svg>
        <div class="tooltip"></div>
    </div>
</template>
<script lang='ts'>
import { defineComponent } from 'vue';
import moment from 'moment';
import * as d3 from 'd3';

import dataJson from '@/assets/aapl-bollinger.json';

interface StatisticsDatum {
    date: Date,
    close: number,
    lower: number,
    middle: number,
    upper: number,
}

interface ComponentData {
    svgSelection?: D3Selection,
    barChart?: D3Selection,
    xAxis?: D3Selection,
    yAxis?: D3Selection,
    slider?: D3Selection,
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
    startDate?: Date,
}

type D3Selection = d3.Selection<any, any, any, any>;

const PARSED_DATA = dataJson.map((elem: StatisticsDatum) => ({ ...elem, date: new Date(elem.date) })) as StatisticsDatum[];
const DAY_IN_MILLISECONDS = 1000 * 60 * 60 * 24;
const MONTH_IN_MILLISECONDS = DAY_IN_MILLISECONDS * 31;
const YEAR_IN_MILLISECONDS = DAY_IN_MILLISECONDS * 365;
const MAXIMUM_BAR_WIDTH = 80;

export default defineComponent({
    name: 'D3Statistics',
    data(): ComponentData {
        return {
            svgSelection: undefined,
            barChart: undefined,
            xAxis: undefined,
            yAxis: undefined,
            slider: undefined,
            tooltip: undefined,
            data: [],
            margin: {
                left: 30,
                right: 50,
                top: 10,
                bottom: 30,
            },
            width: 1140,
            height: 490,
            startDate: undefined,
        };
    },
    mounted() {
        this.data = PARSED_DATA;
        this.startDate = this.data[0].date;
        this.init();
    },
    methods: {
        init(): void {
            this.svgSelection = d3.select('.statistics svg')
                .attr('width', this.width + this.margin.left + this.margin.right)
                .attr('height', this.height + this.margin.top + this.margin.bottom);
            this.xAxis = this.svgSelection.append('g');
            this.yAxis = this.svgSelection.append('g');
            this.barChart = this.svgSelection.append('g');
            this.slider = this.svgSelection.append('g');
            this.renderAxises();
            this.renderBars();
            this.renderSlider();
            this.initTooltip();
        },
        renderAxises(): void {
            this.yAxis
                ?.call(d3.axisLeft(this.y).tickSizeOuter(0).ticks(this.height / 40))
                    .attr('transform', `translate(${this.margin.left},0)`)
                .call(g => {
                    g.selectAll('.tick .copy').remove();
                    g.selectAll('.tick line').clone()
                        .attr('stroke-opacity', 0.15)
                        .attr('x2', this.width + this.xBandwidth)
                        .classed('copy', d => d !== 1);
                });
            this.xAxis?.attr('transform', `translate(${this.margin.left + this.xBandwidth / 2},${this.height})`);
            const [firstDate, lastDate] = this.x.domain();
            const diff = +lastDate - +firstDate;
            const diffMoreThan5Years = diff >= 5 * YEAR_IN_MILLISECONDS;
            const diffMoreThan45Days = diff >= 1.5 * MONTH_IN_MILLISECONDS;
            const diffMoreThan36Hours = diff >= 1.5 * DAY_IN_MILLISECONDS;
            this.xAxis?.call(d3.axisBottom(this.x).tickSizeOuter(0).ticks(this.ticksCount).tickFormat((d: Date) => {
                if (diffMoreThan5Years) return moment(d).locale('ru-RU').format('YYYY');
                if (diffMoreThan45Days) return moment(d).locale('ru-RU').format('MM.YYYY');
                if (diffMoreThan36Hours) return moment(d).locale('ru-RU').format('DD.MM');
                return moment(d).locale('ru-RU').calendar({
                    lastDay: '[Вчера,] HH:mm',
                    sameDay: '[Сегодня,] HH:mm',
                    sameElse: 'HH:mm',
                });
            }));
            this.xAxis?.select('.domain').attr('d', `M${0 - this.xBandwidth / 2},0H${this.width + this.xBandwidth / 2}`);
        },
        renderBars(): void {
            const t = d3.transition().duration(750).ease(d3.easePoly);
            this.barChart?.selectAll('.bar')
                .data(this.data.filter(d => d.date <= new Date()), (data: StatisticsDatum) => data.date)
                .join(
                    enter => enter.append('rect').classed('bar', true).attr('y', this.height).attr('x', data => this.x(data.date)),
                    update => update,
                    exit => exit.call(rect => rect.transition(t).remove().attr('y', this.height).attr('height', 0)),
                )
                .call(rect => rect.transition(t)
                    .attr('width', this.xBandwidth)
                    .attr('height', data => this.height - this.y(data.upper))
                    .attr('x', (elem) => this.x(elem.date))
                    .attr('y', (datum) => this.y(datum.upper))
                    .attr('fill', '#6c8191')
                );
            this.barChart?.selectAll('.bar')
                .on('mouseover', this.mouseoverBar)
                .on('mousemove', this.mousemoveBar)
                .on('mouseleave', this.mouseleaveBar);
            this.barChart?.transition(t).attr('transform', `translate(${this.margin.left}, 0)`);
        },
        renderSlider(): void {
            // todo
        },
        initTooltip(): void {
            this.tooltip = d3.select('.tooltip');
            this.tooltip.classed('tooltip', true);
        },
        mouseoverBar(g: any): void {
            this.tooltip?.style('opacity', 1);
        },
        mousemoveBar(event: MouseEvent): void {
            const data: StatisticsDatum = (event.target as any).__data__;
            this.tooltip?.html(`<div>${moment(data.date).format('DD.MM.YYYY HH:mm:ss')}<br><span class='tooltip-value'>${data.upper.toFixed()}</span> записей</div>`)
                .style('left', (d3.pointer(event, event.target)[0]) - 11 + 'px')
                .style('top', (d3.pointer(event, event.target)[1] - 25) + 'px')
        },
        mouseleaveBar(g: any): void {
            this.tooltip?.style('opacity', 0)
        },
        onButtonClick(days: number): void {
            // filter values between selected date and current date
            const dateOffset = (24 * 60 * 60 * 1000) * days;
            const timeAgoDate = new Date();
            timeAgoDate.setTime(new Date().getTime() - dateOffset);
            this.data = PARSED_DATA.filter((elem: StatisticsDatum) => {
                return elem.date > timeAgoDate && elem.date < new Date();
            });
            this.startDate = timeAgoDate;
            this.renderBars();
            this.renderAxises();
        },
        showAll(): void {
            this.data = PARSED_DATA;
            this.startDate = this.data[0].date;
            this.renderBars();
            this.renderAxises();
        },
    },
    computed: {
        x(): d3.ScaleTime<number, number, never> {
            return d3.scaleUtc()
                .range([0, this.width]).nice()
                .domain([this.startDate ?? +new Date() - 1000, new Date()]);
        },
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
        y(): d3.ScaleLinear<number, number, never> {
            return d3.scaleLinear()
                .range([this.height, this.margin.top]).nice()
                .domain([0, d3.max(this.data, datum => datum.upper) ?? 10]);
        },
        ticksCount(): number {
            return this.width / 120;
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
    &:hover {
        fill: #4fa292;
    }
}
.tooltip {
    position: absolute;
    font-family: 'Roboto';
    font-size: 12px;
    font-weight: 500;
    text-align: center;
    padding: 12px;
    background-color: #e8eaed;
    border: 1px solid #aeb6be;
    border-radius: 5px;
    opacity: 0;
    &-value {
        color: #4fa292;
        font-weight: bold;
    }
}
</style>