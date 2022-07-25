<template>
  <div>
    <h1>web实现浏览器端录音</h1>
    <button @click="start">开始录音</button>
    <button @click="stop">结束录音</button>
    <button @click="play">播放录音</button>
    <button @click="destroy">销毁录音</button>
  </div>
  <div>
    <h2>下载mp3</h2>
    <button @click="download">下载mp3</button>
  </div>
</template>

<script setup>
// import { reactive, ref } from 'vue'
import Recorder from "js-audio-recorder";
var lamejs = require("lamejs");
console.log(lamejs);
let recorder = new Recorder({
  sampleBits: 16, // 采样位数，支持 8 或 16，默认是16
  sampleRate: 16000, // 采样率，支持 11025、16000、22050、24000、44100、48000，根据浏览器默认值，我的chrome是48000
  numChannels: 1,
});
//开始录音
const start = () => {
  Recorder.getPermission().then(
    () => {
      console.log("获取录音权限成功");
    },
    (error) => {
      console.log(`${error.name} : ${error.message}`);
    }
  );
  recorder.start().then(
    () => {
      // 开始录音
    },
    (error) => {
      // 出错了
      console.log(`${error.name} : ${error.message}`);
    }
  );
};
// 停止录音
const stop = () => {
  recorder.stop();
};
// 播放录音
const play = () => {
  recorder.play();
};

// 销毁录音实例，置为null释放资源，fn为回调函数，
const destroy = () => {
  if (recorder) {
    recorder.destroy().then(function () {
      recorder = null;
    });
  }
};
// 下载mp3文件
const download = () => {
  const WavBlob = recorder.getWAV();
  console.log(WavBlob);
  const mp3Blob = convertToMp3(WavBlob); //转换成功的mp3Blob
  recorder.download(mp3Blob, "recorder", "mp3");
  // 发送网络请求到服务器
  // 注意这里的api接口
  //  headers: { 'Content-Type': 'multipart/form-data' },
  // data传入mp3Blol
  let audioFile = new File([mp3Blob], Date.now() + ".mp3", {
    type: mp3Blob.type,
  });
  let fd = new FormData();
  fd.append("filename", audioFile);
  // axios({
  //   headers: {
  //     'Content-Type': 'multipart/form-data'
  //   },
  //   method: 'post',
  //   url: '/file/upload',
  //   data: fd
  // })
  // let result = await apiUpload(fd)
};
// 通过lamejs把wav转换成mp3Blol
const convertToMp3 = (wavDataView) => {
  // 获取wav头信息
  const wav = lamejs.WavHeader.readHeader(wavDataView); // 此处其实可以不用去读wav头信息，毕竟有对应的config配置
  const { channels, sampleRate } = wav;
  const mp3enc = new lamejs.Mp3Encoder(channels, sampleRate, 128);
  // 获取左右通道数据
  const result = recorder.getChannelData();
  const buffer = [];

  const leftData =
    result.left &&
    new Int16Array(result.left.buffer, 0, result.left.byteLength / 2);
  const rightData =
    result.right &&
    new Int16Array(result.right.buffer, 0, result.right.byteLength / 2);
  const remaining = leftData.length + (rightData ? rightData.length : 0);

  const maxSamples = 1152;
  for (let i = 0; i < remaining; i += maxSamples) {
    const left = leftData.subarray(i, i + maxSamples);
    let right = null;
    let mp3buf = null;

    if (channels === 2) {
      right = rightData.subarray(i, i + maxSamples);
      mp3buf = mp3enc.encodeBuffer(left, right);
    } else {
      mp3buf = mp3enc.encodeBuffer(left);
    }

    if (mp3buf.length > 0) {
      buffer.push(mp3buf);
    }
  }

  const enc = mp3enc.flush();

  if (enc.length > 0) {
    buffer.push(enc);
  }

  return new Blob(buffer, { type: "audio/mp3" });
};
</script>

<style lang="scss" scoped></style>
