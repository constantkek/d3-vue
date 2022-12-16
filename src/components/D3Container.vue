<template>
  <div>
    <div class="d3-container">
      <svg></svg>
    </div>
    <div id="data">
      <ul></ul>
    </div>
  </div>
</template>

<script lang='ts'>
import { defineComponent } from "vue";
import * as d3 from 'd3';
import * as d3Scale from 'd3-scale';

const PEOPLE = [
  {
    id: '1',
    name: 'Andy',
    age: 18,
  },
  {
    id: '2',
    name: 'Sarah',
    age: 15,
  },
  {
    id: '3',
    name: 'Lily',
    age: 23,
  },
  {
    id: '4',
    name: 'John',
    age: 16,
  },
  {
    id: '5',
    name: 'Peter',
    age: 21,
  },
];

const MARGIN_TOP = 40;
const MARGIN_BOTTOM = 20;
const CHART_HEIGHT = 450 - MARGIN_TOP - MARGIN_BOTTOM;
const CHART_WIDTH = 800;

interface People {
  id: string,
  name: string,
  age: number
}

interface DataType {
  selectedIds: string[];
  testData: People[],
  svgContainer?: d3.Selection<any, any, HTMLElement, any>,
  chart?: d3.Selection<any, People, any, People>,
  x: d3.ScaleBand<string>,
  y: d3.ScaleLinear<number, number, number>,
}

export default defineComponent({
  name: "D3Container",
  data() {
    return {
      selectedIds: [...PEOPLE.map(data => data.id)],
      svgContainer: undefined,
      testData: PEOPLE,
      chart: undefined,
      x: d3Scale.scaleBand(),
      y: d3Scale.scaleLinear(),
    } as DataType;
  },
  mounted(): void {
    this.initSvg();
    this.renderChart();
    this.initControls();
  },
  computed: {
    selectedData(): People[] {
      return this.testData.filter(data => this.selectedIds.includes(data.id))
    },
  },
  methods: {
    initSvg() {
      this.svgContainer = d3.select('svg').attr('width', '100%').attr('height', CHART_HEIGHT + MARGIN_BOTTOM + MARGIN_TOP);
      this.chart = this.svgContainer.append('g');
    },
    initDomains() {
      this.x = d3Scale.scaleBand().rangeRound([0, CHART_WIDTH]).padding(.15);
      this.y = d3Scale.scaleLinear().range([CHART_HEIGHT, MARGIN_TOP]);

      this.x.domain(this.selectedData.map(data => data.name));
      this.y.domain([0, d3.max(this.selectedData, data => data.age) ?? 1]);
    },
    renderChart() {
      this.initDomains();
      this.chart
        ?.selectAll('.bar').data([]).exit().remove();
      this.chart
        ?.selectAll('.bar')
        .data(this.selectedData, data => (data as People).id)
        .enter().append('rect')
        .classed('bar', true)
        .attr('width', this.x.bandwidth())
        .attr('height', data => CHART_HEIGHT - this.y(data.age))
        .attr('x', data => this.x(data.name) ?? 0)
        .attr('y', data => this.y(data.age));
      this.chart
        ?.selectAll('.bar-label').data([]).exit().remove();
      this.chart
        ?.selectAll('.bar-label')
        .data(this.selectedData, data => (data as People).id)
        .enter().append('text')
        .classed('bar-label', true)
        .text(data => data.age)
        .attr('x', data => (this.x(data.name) ?? 0) + this.x.bandwidth() / 2)
        .attr('y', data => this.y(data.age) - 20);
      this.svgContainer?.select('.bottom-axis').remove();
      this.svgContainer?.append('g').call(d3.axisBottom(this.x).tickSizeOuter(0)).attr('transform', `translate(0, ${CHART_HEIGHT})`).attr('color', '#6f6f6f').classed('bottom-axis', true);
    },
    initControls() {
      const listItems = d3.select('#data').select('ul').selectAll('li').data(this.testData).enter().append('li');
      listItems.append('span').text(data => data.name);
      listItems
        .append('input')
        .attr('type', 'checkbox')
        .attr('checked', data => this.selectedIds.includes(data.id) || null)
        .on('change', (event) => {
          const { id } = event.target.__data__;
          if (this.selectedIds.includes(id)) {
            this.selectedIds = this.selectedIds.filter(selectedId => selectedId !== id);
          } else {
            this.selectedIds.push(id);
          }
          this.renderChart();
        });
    },
  },
});
</script>

<style lang="scss">
.d3-container {
  padding: 15px;
  width: 800px;
  box-shadow: 0px 0px 10px 0px rgba(0, 0, 0, 0.5);
  border-radius: 4px;
}
.bar {
  fill: steelblue;
}
.bar-label {
  fill: #3f3f3f;
  font: bold 16px "Segoe UI";
  text-anchor: middle;
}
</style>
