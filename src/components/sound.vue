<template>
  <div class="sound">
        <button @click="play()">Play</button>
        <button @click="save()">Save</button>
        <button @click="loadWav()">Load WAV</button>
        <audio controls>
            <source ref="source" src="@/assets/sound/a.wav" />
        </audio>
        <canvas ref="chart" height="200" width="800"/>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import Chart,{ prototype } from 'chart.js';
import * as tf from '@tensorflow/tfjs';
import { constant } from '@tensorflow/tfjs-layers/dist/exports_initializers';
import axios from 'axios';

// import ChartLine from './stand-alone/chart-line/index.vue';

@Component
export default class Sound extends Vue {
    protected rate: number = 44100;
    protected bufSize: number = this.rate * 3;

    protected mounted() {
        const r = new FileReader();
        const source = this.$refs.source as HTMLSourceElement;
        
        axios.get(source.src, { responseType:"blob" }).then((response) => {
            const data = response && response.data;
            const blob = new Blob([data]);
            const r = new FileReader();
            r.readAsArrayBuffer(data);
            r.onload = () => {
                const buf = new Int16Array(r.result).slice(44);
                this.drawChart(buf);
                console.log(buf);
            };
        });
    }

    protected loadWav() {
        const source = this.$refs.source as HTMLSourceElement;
        console.log(source.attributes);
    }

    protected play() {
        const base64string = this.createWave().toString();
        const snd = new Audio("data:audio/wav;base64," + base64string);
        snd.play();
    }

    protected save() {
        const wav = this.createWave();
        wav.save('sound.wav');
    }

    protected createWave(): Wave {
        const amplitude = 32000;
        const freqHz = 250;

        const buf: Int16Array = new Int16Array(this.bufSize);

        for (let i = 0; i < this.bufSize; i++) {
            buf[i] = amplitude * Math.sin(i * 2 * Math.PI * freqHz / this.rate);
        }

        const wav = new Wave(buf, buf.length);

        this.drawChart(buf);

        return wav;
    }

    protected drawChart(data: Int16Array, start: number = 0, end: number | undefined = undefined) {
        const ctx = (this.$refs.chart as HTMLCanvasElement).getContext('2d');
        if (!ctx) { return; }
        const slice: number[] = Array.prototype.slice.call(data.slice(start, end)).map((v: number) => v / 32000);
        const labels = slice.map((v) => v.toFixed(2));
        console.log(slice, labels);

        new Chart(ctx, {
            type: 'line',
            data: {
                labels: labels,
                datasets: [
                    {
                        data: slice,
                    },
                ],
            },
            options: {
                responsive: false,
                elements: {
                    point: {
                        radius: 0,
                    },
                },
                legend: {
                    display: false,
                },
            },
        });
    }
}

class Wave {
    private HEADER_SIZE = 44;
    private buf?: ArrayBuffer;
    private dv?: DataView;

    private p: number = 0;

    constructor (data: Int16Array, numFrames: number = 1, numChannels: number = 1,
                 sampleRate: number = 44100, bytesPerSample: number = 2) {

        this.buf = new ArrayBuffer(this.HEADER_SIZE + numFrames * bytesPerSample);
        this.dv = new DataView(this.buf)

        this.buildHeader(numFrames, numChannels, sampleRate, bytesPerSample);
        this.writeData(data);
    }

    public buildHeader(numFrames: number = 1, numChannels: number = 1, sampleRate: number = 44100, 
                       bytesPerSample: number = 2) {

        const blockAlign = numChannels * bytesPerSample;
        const byteRate = sampleRate * blockAlign;
        const dataSize = numFrames * blockAlign;

        this.writeString('RIFF');              // ChunkID
        this.writeUint32(dataSize + 36);       // ChunkSize
        this.writeString('WAVE');              // Format
        this.writeString('fmt ');              // Subchunk1ID
        this.writeUint32(16);                  // Subchunk1Size
        this.writeUint16(1);                   // AudioFormat
        this.writeUint16(numChannels);         // NumChannels
        this.writeUint32(sampleRate);          // SampleRate
        this.writeUint32(byteRate);            // ByteRate
        this.writeUint16(blockAlign);          // BlockAlign
        this.writeUint16(bytesPerSample * 8);  // BitsPerSample
        this.writeString('data');              // Subchunk2ID
        this.writeUint32(dataSize); // Subchunk2Size
    }

    public writeData(data: Int16Array) {
        if (!this.dv) { return ; }
        for (let i = 0; i < data.length; i++) {
            this.writeUint16(data[i]);
        }
    }

    public toString(): string {
        if (!this.buf) { return '';}

        let binary = '';
        const bytes = new Uint8Array(this.buf);
        const len = bytes.byteLength;
        for (var i = 0; i < len; i++) {
            binary += String.fromCharCode( bytes[ i ] );
        }
        return window.btoa( binary );
    }

    public save(fileName: string) {
        if (!this.buf) { return ; }
        const blob = new Blob([new Uint8Array(this.buf)]);
        if (window.navigator.msSaveOrOpenBlob) {
            window.navigator.msSaveOrOpenBlob(blob, fileName);
        } else {
            var a = document.createElement("a"),
                    url = URL.createObjectURL(blob);
            a.href = url;
            a.download = fileName;
            document.body.appendChild(a);
            a.click();
            setTimeout(function() {
                document.body.removeChild(a);
                window.URL.revokeObjectURL(url);  
            }, 0); 
        }
    }

    private writeString(s: string) {
        if (!this.dv) { return ; }
        for (let i = 0; i < s.length; i++) {
            this.dv.setUint8(this.p + i, s.charCodeAt(i));
        }
        this.p += s.length;
    }

    private writeUint32(d: number) {
        if (!this.dv) { return ; }
        this.dv.setUint32(this.p, d, true);
        this.p += 4;
    }

    private writeUint16(d: number) {
        if (!this.dv) { return ; }
        this.dv.setUint16(this.p, d, true);
        this.p += 2;
    }
}

interface WaveOptions {
    numFrames: number;
    numChannels: number;
    sampleRate: number;
    bytesPerSample: number;
}
</script>

<style scoped lang="scss">
</style>
