<template>
  <div class="polynome">
    <chart-line :values="values.ys"/>
    <p>{{getGeneratedDesc()}}</p>
    <chart-line :values="values2"/>
    <p>{{getCalculatedDesc()}}</p>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import Chart,{ prototype } from 'chart.js';
import * as tf from '@tensorflow/tfjs';
import { constant } from '@tensorflow/tfjs-layers/dist/exports_initializers';

import ChartLine from './stand-alone/chart-line/index.vue';

@Component({
    components: {
        ChartLine,
    }
})
export default class Polynome extends Vue {
    protected learningRate = 0.5;
    protected optimizer = tf.train.sgd(this.learningRate);

    protected values: {xs: number[], ys: number[]} = {xs: [], ys: []};
    protected values2: number[] = [];

    protected ta: number = Math.random();
    protected tb: number = Math.random();
    protected tc: number = Math.random();
    protected td: number = Math.random();

    protected a = tf.variable(tf.scalar(Math.random()));
    protected b = tf.variable(tf.scalar(Math.random()));
    protected c = tf.variable(tf.scalar(Math.random()));
    protected d = tf.variable(tf.scalar(Math.random()));

    protected mounted() {
        this.values = this.generateData();

       const xs = tf.tensor2d(this.values.xs, [this.values.xs.length, 1]);
       const ys = tf.tensor2d(this.values.ys, [this.values.ys.length, 1]);

        console.log('a: ' + this.a.toString());
        this.train(xs, ys);
        
        let output: Float32Array = this.predict(xs).dataSync() as Float32Array;
        const res: number[] = [];
        output.forEach((v: number) => res.push(v));
        this.values2 = res;
        console.log(output.map((v: number) => v));
        console.log(res);
        // this.values2 = output;
        console.log('a: ' + this.a.toString());

    //     const model = tf.sequential();
    //   model.add(tf.layers.dense({units: 1, inputShape: [1]}));

    //   // Prepare the model for training: Specify the loss and the optimizer.
    //   model.compile({loss: 'meanSquaredError', optimizer: 'sgd'});

    //   // Generate some synthetic data for training.
    //   const xs = tf.tensor2d(this.values.xs, [this.values.xs.length, 1]);
    //   const ys = tf.tensor2d(this.values.ys, [this.values.ys.length, 1]);

    //   // Train the model using the data.
    //   model.fit(xs, ys, {epochs: 10}).then(() => {
    //     // Use the model to do inference on a data point the model hasn't seen before:
    //     // Open the browser devtools to see the output
    //     const output = model.predict(tf.tensor2d([5], [1, 1]));
    //     console.log(output.toString());
    //   });
    }

    protected train(xs: tf.Tensor2D, ys: tf.Tensor2D, numIterations = 100000) {

        const learningRate = 1e-7;
        const epsilon = 0.9;
        const optimizer = tf.train.sgd(learningRate);

        for (let iter = 0; iter < numIterations; iter++) {
            optimizer.minimize(() => {
                const predsYs = this.predict(xs);
                const l = this.loss(predsYs, ys);
                console.log(iter, l.toString());
                return l;
            });
        }
    }

    protected generateData(): {xs: number[], ys: number[]} {
        const res: {xs: number[], ys: number[]} = {xs: [], ys: []};

        for (let x: number = 0; x < 20; x++) {
            res.xs.push(x);
            res.ys.push(this.ta * x ^ 3 + this.tb * x ^ 2 + this.tc * x + this.td);
        }

        return res;
    }

    protected predict(x:tf.Tensor2D) {
        // y = a * x ^ 3 + b * x ^ 2 + c * x + d
        return tf.tidy(() => {
            return this.a.mul(x.pow(tf.scalar(3))) // a * x^3
            .add(this.b.mul(x.square())) // + b * x ^ 2
            .add(this.c.mul(x)) // + c * x
            .add(this.d); // + d
        });
    }

    protected loss(predictions: any, labels: any): any {
        const mse = predictions.sub(labels).square().mean();
        return mse;
    }

    protected getGeneratedDesc(): string {
        return `a: ${this.ta.toFixed(3)} b: ${this.tb.toFixed(3)} c: ${this.tc.toFixed(3)} d: ${this.td.toFixed(3)}`;
    }

    protected getCalculatedDesc(): string {
        return `a: ${this.a.get().toFixed(3)} b: ${this.b.get().toFixed(3)} c: ${this.c.get().toFixed(3)} d: ${this.d.get().toFixed(3)}`;
    }
}
</script>

<style scoped lang="scss">
</style>
