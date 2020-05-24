<template>
  <div>
    {{ file.name }}
    <div class="file-box">
      <input
        id="file"
        class="file"
        type="file"
        @change="handleFileChange"
      >
    </div>
    <button @click="handleUpload">上传</button>
    {{ hashProgress }}
  </div>
</template>

<script>
import { FILE } from './helper/constants';
import SparkMD5 from 'spark-md5';
// import hashWorker from './helper/hash.worker';

export default {
  name: 'Upload',
  data () {
    return {
      chunks: [],
      file: '',
      hash: null,
      hashProgress: 0
    }
  },
  methods: {
    blobToString (blob) {
      return new Promise((resolve) => {
        const reader = new FileReader();

        reader.onload = function () {
          const ret = reader.result.split('')
            .map(v => v.charCodeAt()) // 二进制 => Unicode编码
            .map(v => v.toString(16).toUpperCase()) // Unicode编码 => 16进制字符串
            .map(v => v.padStart(2, '0'))
            .join(' ');

          resolve(ret);
        }
        reader.readAsBinaryString(blob);
      });
    },
    async isGif (file) {
      const ret = await this.blobToString(file.slice(0, 6));

      return (ret === '47 49 46 38 39 611' || ret === '47 49 46 38 37 61');
    },
    async isPng (file) {
      const ret = await this.blobToString(file.slice(0, 8));

      return ret === '89 50 4E 47 0D 0A 1A 0A';
    },
    async isJpg (file) {
      const len = file.size;
      const start = await this.blobToString(file.slice(0, 8));
      const tail = await this.blobToString(file.slice(-2, len));

      return start === 'FF D8' && tail === 'FF D9';
    },
    isImage (file) {
      // 通过16进制文件流来判断文件类型
      // 文件通过file.type 和 file.name来判断文件类型不准确
      return Promise.all([this.isGif(file), this.isPng(file), this.isJpg(file)])
        .then(([isgif, ispng, isjpg]) => isgif || ispng || isjpg);
    },
    handleFileChange (event) {
      const [file] = event.target.files;

      if (file.size > FILE.CHUNK_SIZE) {
        console.log('文件超过上限1M');
        return;
      }

      this.isImage(file)
        .then((isimg) => {
          if (!isimg) {
            console.log('请选择正确的图片格式');
          } else {
            this.file = file;
          }
        });

      return false
    },
    mergeRequest () {
      return this.$axios.post('/merge', {
        ext: this.ext(this.file.name),
        size: FILE.CHUNK_SIZE,
        hash: this.hash
      });
    },
    ext (filename) {
      return filename.split('.').pop();
    },
    createFileChunk (file, size = FILE.CHUNK_SIZE) {
      const chunks = [];
      let current = 0;

      while (current < file.size) {
        chunks.push({ index: current, file: file.slice(current, current + size) });
        current += size;
      }
      return chunks;
    },
    blobToData (blob) {
      return new Promise((resolve) => {
        const reader = new FileReader();

        reader.onload = function () {
          resolve(reader.result)
        }
        reader.onerror = function () {
          console.warn('file reader fail!');
        }
        reader.readAsArrayBuffer(blob)
      });
    },
    async calculateHash (chunks) {
      let l = chunks.length;
      const spark = new SparkMD5.ArrayBuffer();

      while (l--) {
        const ret = await this.blobToData(chunks[l].file);

        spark.append(ret);
      }
      return spark.end();
    },
    // calculateHashWorker (chunks) {
    // return new Promise((resolve) => {
    // web worker 需要同源策略，而且worker不能读取本地文件
    // vue中需要配置worker-loader插件，保证引用的路径
    // const worker = new hashWorker();

    // worker.postMessage({ chunks });
    // worker.onmessage = e => {
    //   const { progress, hash } = e.data;

    //   this.hashProgress = Number(progress.toFixed(2));
    //   if (hash) {
    //     worker.terminate();
    //     resolve(hash);
    //   }
    // }
    // });
    // },
    calculateHashIdle (chunks) {
      return new Promise((resolve) => {
        const spark = new SparkMD5.ArrayBuffer();
        let count = 0;
        const appendToSpark = async (file) => {
          return new Promise((resolve) => {
            const reader = new FileReader();

            reader.readAsArrayBuffer(file);
            reader.onload = function (e) {
              spark.append(e.target.result);
              resolve();
            }
          });
        }
        const workLoop = async (deadline) => {
          // 有任务，并且当前帧还没有结束
          while (count < chunks.length && deadline.timeRemaining() > -1) {
            await appendToSpark(chunks[count].file);
            count++;
            if (count < chunks.length) {
              // 计算中
              this.hashProgress = Number(((100 * count) / chunks.length).toFixed(2));
            } else {
              // 计算完毕
              this.hashProgress = 100;
              resolve(spark.end());
            }
          }

          console.log(`浏览器有任务拉，开始计算${count}个，等待下次浏览器空闲`);
          window.requestIdleCallback(workLoop);
        }

        window.requestIdleCallback(workLoop);
      });
    },
    calculateHashSample () {
      return new Promise((resolve) => {
        const spark = new SparkMD5.ArrayBuffer();
        const reader = new FileReader();
        const file = this.file;
        const size = file.size;
        const offset = FILE.CHUNK_SIZE;
        let chunks = [file.slice(0, offset)];
        let current = offset;

        while (current < size) {
          if (current + offset > size) {
            chunks.push(file.slice(current));
          } else {
            const mid = current + offset / 2;
            const end = current + offset;

            chunks.push(file.slice(current, current + 2));
            chunks.push(file.slice(mid, mid + 2));
            chunks.push(file.slice(end - 2, end));
          }
          current += offset;
        }

        reader.readAsArrayBuffer(new Blob(chunks));

        reader.onload = e => {
          spark.append(e.target.result);
          console.log(e)
          this.hashProgress = 100;
          resolve(spark.end());
        }
      });
    },
    async handleUpload () {
      if (!this.file) {
        console.log('files must existed!')
      }

      let chunks = this.createFileChunk(this.file);

      // md5计算hash
      // 直接使用md5计算哈希，大文件会卡顿
      // this.hash = await this.calculateHash(chunks);

      // web-worker 开启子线程，防止主线程卡顿
      // vue中使用web-worker需要使用worker-loader辅助
      // this.hash = await this.calculateHashWorker(chunks);

      // 利用浏览器空闲时间计算hash
      // 在事件循环空闲时即将被调用的函数
      // this.hash = await this.calculateHashIdle(chunks);

      // 抽样哈希
      // 牺牲一定的准确性，来换取更高的哈希效率
      this.hash = await this.calculateHashSample();

      const { uploaded, uploadedList } = await this.$axios.post('/check', {
        ext: this.ext(this.file.name),
        hash: this.hash
      });

      if (uploaded) {
        console.log('秒传：上传成功');
        return;
      }

      this.chunks = chunks.map((chunk, index) => {
        const chunkName = this.hash + '-' + index;

        return {
          hash: this.hash,
          chunk: chunk.file,
          name: chunkName,
          index,
          progress: uploadedList.index(chunkName) > -1 ? 100 : 0,
        }
      });

      await this.uploadChunks(uploadedList);
    },
    async uploadChunks (uploadedList = []) {
      const list = this.chunks
        .filter(chunk => uploadedList.indexOf(chunk.name) === -1)
        .map(({ chunk, name, hash, index }) => {
          const form = new FormData();
          form.append('chunkname', name);
          form.append('ext', this.ext(this.file.name));
          form.append('hash', hash);
          form.append('file', chunk);

          return { form, index, error: 0 }
        });

      try {
        await this.sendRequest([...list], 4);
        if (uploadedList.length + list.length === this.chunks.length) {
          await this.mergeRequest();
        }
      } catch (e) {
        console.log('上传出现问题了');
      }
    }
  }
}
</script>

<style scoped>
.file-box {
  border-radius: 6px;
  width: 100px;
  height: 100px;
  background-color: #3c3c3c;
}

.file {
  width: 100%;
  height: 100%;
  opacity: 0;
}
</style>
