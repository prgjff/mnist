<template>
  <div class="mnist">
    <div class="col">
        <button @click="train()">Train</button>
        <p>{{status}}</p>
        <canvas id="chart-loss"></canvas>
        <canvas id="chart-acc"></canvas>
    </div>
    <div class="col">
        <button @click="test()">Test</button>
        <canvas height="28px" width="28px" id="canvas"></canvas>
        <p>{{result}}</p>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import Chart from 'chart.js';
import * as tf from '@tensorflow/tfjs';
import { constant } from '@tensorflow/tfjs-layers/dist/exports_initializers';

@Component
export default class Mnist extends Vue {
    protected mnistData = require('./data/data.js');

  protected LEARNING_RATE = 0.15;
  protected optimizer = tf.train.sgd(this.LEARNING_RATE);
  protected BATCH_SIZE = 64;
  protected TRAIN_BATCHES = 100;

  protected TEST_BATCH_SIZE = 1000;
  protected TEST_ITERATION_FREQUENCY = 5;

  protected model?: tf.Model;
  protected data?: any;

  protected status: string = '';
  protected result: string = '';

  protected history: {loss: number[], acc: number[]} = {loss: [], acc: []};

  protected async mounted() {
    this.status = 'data loading';
    this.model = this.initModel();
    this.data = new this.mnistData.MnistData();
    await this.data.load();
    this.status = 'data loaded';
  }

  protected async train() {
      if (!this.model) { return; }
      if (!this.data) { return; }

      this.history = {loss: [], acc: []};

      for (let i = 0; i < this.TRAIN_BATCHES; i++) {
        const batch = this.data.nextTrainBatch(this.BATCH_SIZE);

        let testBatch;
        let validationData;

        if (i % this.TEST_ITERATION_FREQUENCY === 0) {
          testBatch = this.data.nextTestBatch(this.TEST_BATCH_SIZE);
          validationData = [
            testBatch.xs.reshape([this.TEST_BATCH_SIZE, 28, 28, 1]), testBatch.labels,
          ];
        }

        const history = await this.model.fit(
            batch.xs.reshape([this.BATCH_SIZE, 28, 28, 1]),
            batch.labels,
            // {
            //   batchSize: this.BATCH_SIZE,
            //   validationData,
            //   epochs: 1
            // }
        );
        this.history.loss.push(history.history.loss[0] as number);
        this.history.acc.push(history.history.acc[0] as number);
        const loss = history.history.loss[0];
        const accuracy = history.history.acc[0];
        this.status = `loss: ${(loss as number).toFixed(2)} accuracy: ${(accuracy as number).toFixed(2)}`;
      }
      this.drawChart('chart-loss', this.history.loss);
      this.drawChart('chart-acc', this.history.acc);
  }

  protected initModel(): tf.Model {
    const result = tf.sequential();

    result.add(tf.layers.conv2d({
      inputShape: [28, 28, 1],
      kernelSize: 5,
      filters: 8,
      strides: 1,
      activation: 'relu',
      kernelInitializer: 'VarianceScaling',
    }));

    result.add(tf.layers.maxPooling2d({
      poolSize: [2, 2],
      strides: [2, 2],
    }));

    result.add(tf.layers.conv2d({
      kernelSize: 5,
      filters: 16,
      strides: 1,
      activation: 'relu',
      kernelInitializer: 'VarianceScaling',
    }));

    result.add(tf.layers.maxPooling2d({
      poolSize: [2, 2],
      strides: [2, 2],
    }));

    result.add(tf.layers.flatten());

    result.add(tf.layers.dense({
      units: 10,
      kernelInitializer: 'VarianceScaling',
      activation: 'softmax',
    }));

    result.compile({
      optimizer: this.optimizer,
      loss: 'categoricalCrossentropy',
      metrics: ['accuracy'],
    });

    return result;
  }

  protected draw(image: any, canvas: any) {
    const [width, height] = [28, 28];
    canvas.width = width;
    canvas.height = height;
    const ctx = canvas.getContext('2d');
    const imageData = new ImageData(width, height);
    const data = image.dataSync();
    for (let i = 0; i < height * width; ++i) {
        const j = i * 4;
        imageData.data[j + 0] = data[i] * 255;
        imageData.data[j + 1] = data[i] * 255;
        imageData.data[j + 2] = data[i] * 255;
        imageData.data[j + 3] = 255;
    }
    ctx.putImageData(imageData, 0, 0);
    }

    protected test() {
        const batch = this.data.nextTestBatch(this.TEST_BATCH_SIZE);
        const image = batch.xs.slice([0, 0], [1, batch.xs.shape[1]]);
        const canvas = document.getElementById('canvas');

        this.draw(image.flatten(), canvas);
        if (this.model) {
            const axis = 1;
            const validationData = batch.xs.reshape([this.TEST_BATCH_SIZE, 28, 28, 1]);
            const output: any = this.model.predict(validationData);
            const predictions = Array.from(output.argMax(axis).dataSync());
            this.result = predictions[0].toString();
        }
    }

    protected drawChart(chartID: string, values: number[]) {
        const chart: any = document.getElementById(chartID);
        if (!chart) { return; }
        const ctx = chart.getContext('2d');
        const myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: values.map((v: number) => v.toFixed(2).toString()),
                datasets: [{
                    data: values,
                    borderWidth: 1,
                    borderColor: 'blue',
                    fill: false,
                }],
            },
            options: {
                legend: {
                    display: false,
                },
            },
        });
    }
}
</script>

<style scoped lang="scss">
.mnist {
    display: flex;
    
    .col {
        display: flex;
        flex-direction: column;
        flex: 1 1 10em;
        align-items: center;
    }
}
#chart-loss, #chart-acc {
    height: 200px !important;
}

.canvas {
    height: 28px !important;
    width: 28px !important;
}

button {
    margin: 0.5em;
    padding: 0.25em 0.5em;
    width: 5em;
}
</style>
