<template>
  <canvas :id="'chart-' + _uid" :height="height" :width="width"></canvas>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';
import Chart from 'chart.js';

@Component({
    name: 'chart-line',
})
export default class ChartLine extends Vue {
    @Prop({default: []}) public values!: number[];
    @Prop({default: '250'}) public height: string = '250';
    @Prop({default: '1000'}) public width: string = '1000';

    protected chart?: Chart = undefined;

    @Watch('values')
    protected build() {
        console.log('build');
        const chart: any = document.getElementById('chart-' + this._uid);
        if (!chart) { return; }
        const ctx = chart.getContext('2d');
        
        this.chart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: this.values.map((v: number) => v.toFixed(2).toString()),
                datasets: [{
                    data: this.values,
                    borderWidth: 1,
                    borderColor: 'blue',
                    fill: false,
                }],
            },
            options: {
                responsive: false,
                legend: {
                    display: false,
                },
            },
        });
    }
}
</script>

<style scoped lang="scss">
.chart {
    
}
</style>
