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
    svg?: SVGSVGElement,
    svgSelection?: D3Selection,
    barChart?: D3Selection,
    xAxis?: D3Selection,
    yAxis?: D3Selection,
    slider?: D3Selection,
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
const PARSED_DATA = dataJson.map((elem: StatisticsDatum) => ({ ...elem, date: new Date(elem.date) })) as StatisticsDatum[]
const DAY_IN_MILLISECONDS = 1000 * 60 * 60 * 24;
const MONTH_IN_MILLISECONDS = DAY_IN_MILLISECONDS * 31;

export default defineComponent({
    name: 'D3Statistics',
    data(): ComponentData {
        return {
            svgSelection: undefined,
            barChart: undefined,
            xAxis: undefined,
            yAxis: undefined,
            slider: undefined,
            data: [],
            margin: {
                left: 30,
                right: 50,
                top: 10,
                bottom: 30,
            },
            width: 1400,
            height: 450,
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
            this.svg = this.svgSelection.node();
        },
        renderAxises(): void {
            const t = d3.transition().duration(500);
            this.yAxis
                ?.call(d3.axisLeft(this.y).tickSizeOuter(0).ticks(this.height / 40))
                    .attr('transform', `translate(${this.margin.left},0)`)
                .call(g => {
                    g.selectAll(".tick .copy").remove();
                    g.selectAll(".tick line").clone()
                        .attr("stroke-opacity", d => d === 1 ? 1 : 0.15)
                        .attr("x2", this.width)
                        .classed('copy', d => d !== 1);
                });
            this.xAxis?.attr('transform', `translate(${this.margin.left + this.xBandwidth / 2},${this.height})`);
            const [firstDate, lastDate] = this.x.domain();
            const diff = +lastDate - +firstDate;
            this.xAxis?.call(d3.axisBottom(this.x).tickSizeOuter(0).ticks(this.ticksCount).tickFormat((d: Date) => {
                debugger;
                if (diff >= 1.5 * MONTH_IN_MILLISECONDS) return moment(d).locale('ru-RU').format('MM.YYYY');
                if (diff >= 1.5 * DAY_IN_MILLISECONDS) return moment(d).locale('ru-RU').format('DD.MM');
                return moment(d).locale('ru-RU').format('HH:mm');
            }));
            this.xAxis?.select('.domain').attr('d', `M${0 - this.xBandwidth / 2},0H${this.width + this.xBandwidth / 2}`);
        },
        renderBars(): void {
            const t = d3.transition().duration(500);
            this.barChart?.selectAll('.bar')
                .data(this.data.filter(d => d.date <= new Date()), (data: StatisticsDatum) => data.date)
                .join(
                    enter => enter.append('rect').classed('bar', true).attr('y', this.height).attr('x', data => this.x(data.date)),
                    update => update,
                    exit => exit.call(rect => rect.transition(t).remove().attr('y', this.height).attr('height', 0)),
                )
                .call(text => text.transition(t)
                    .attr('width', this.xBandwidth)
                    .attr('height', data => this.height - this.y(data.upper))
                    .attr('x', (elem) => this.x(elem.date))
                    .attr('y', (datum) => this.y(datum.upper))
                    .attr('fill', '#6c8191'));
            this.barChart?.transition(t).attr('transform', `translate(${this.margin.left}, 0)`);
        },
        renderSlider(): void {
            // todo
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
            if (this.data.length) {
                this.renderAxises();
            }
        },
        showAll(): void {
            this.data = PARSED_DATA;
            this.startDate = this.data[0].date;
            this.renderBars();
            this.renderAxises();
        }
    },
    computed: {
        x(): d3.ScaleTime<number, number, never> {
            return d3.scaleUtc()
                .range([0, this.width]).nice()
                .domain([this.startDate ?? +new Date() - 1000, new Date()]);
        },
        xBandwidth(): number {
            return this.width / this.data.length / 2;
        },
        y(): d3.ScaleLinear<number, number, never> {
            return d3.scaleLinear()
                .range([this.height, this.margin.top]).nice()
                .domain([0, d3.max(this.data, datum => datum.upper) ?? 1]);
        },
        ticksCount(): number {
            return this.width / 80 > this.data.length ? this.data.length : this.width / 80;
        },
    }
});
</script>
<style>
.statistics {
    background-color: #f5f6f7;
    width: min-content;
    padding: 15px 20px;
}
.bar {
    cursor: pointer;
}
</style>