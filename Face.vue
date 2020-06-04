<template>
  <div class="videos">
    <div class="left">
      <div v-show="!video_show" class="video-before">
        <div class="tip">
          <p style="font-size: 18px;">
            为防范身份信息冒用，需进行人脸识别验证
          </p>
          <div class="type-tip">
            <div>
              <icon-font type="icon-right1" ostyle="width: 16px; height: 16px;" />
              <span>保持正对摄像头</span>
            </div>
            <div>
              <icon-font type="icon-right1" ostyle="width: 16px; height: 16px;" />
              <span>保持光线充足</span>
            </div>
            <div>
              <icon-font type="icon-error" ostyle="width: 16px; height: 16px;" />
              <span>不要戴眼镜</span>
            </div>
            <div>
              <icon-font type="icon-error" ostyle="width: 16px; height: 16px;" />
              <span>不要戴帽子</span>
            </div>
          </div>
        </div>
        <img class="before-img" src="@/assets/images/scan.png?inline">
        <Button type="primary" size="large" style="width: 260px;" @click="startPhoto">
          {{ isClick ? '重新拍摄' : '开始拍摄' }}
        </Button>
        <div class="btn-tip">
          请确保由被授权人“ <span>{{ name }}</span>”本人操作
        </div>
      </div>
      <div v-show="video_show" class="video-after">
        <video ref="video" class="video">
          您的浏览器不支持拍照，请使用chrome浏览器
        </video>
        <!--隐藏掉为了发送照片-->
        <canvas v-show="canvas_show" id="canvas" ref="canvas" :width="width" :height="height" />
        <div class="btn-wrap">
          <icon-font type="icon-camera" ostyle="width: 50px; height: 50px;" @click.native="takePhoto" />
          <!-- <img src="@/assets/images/camera.svg" alt="" @click="takePhoto"> -->
        </div>
      </div>
    </div>
    <div class="right">
      <span class="result-title">
        拍摄结果：
      </span>
      <img v-if="imgSrc" class="result-img" alt="照片" :src="imgSrc">
      <div v-else class="result-img" />
      <div class="result-info">
        <p class="result-title">
          被授权人信息：
        </p>
        <ul style="line-height: 26px; margin-top: 6px;">
          <li><span>姓名：</span><span>{{ name }}</span></li>
          <li><span>身份证：</span><span>{{ cardNo }}</span></li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
/**
 * Face compontent
 * @props type: the type of url 人脸对比： 'facecompare' 人证核身：facevalidate
 * @props size: video size 'small', 'middle', 'large'
 * @props name: the name for show
 * @props cardNo: the cardNo for show
 * @event validateSuc: back result to parent
 */
const sizeType = {
  small: {
    width: 320,
    height: 240
  },
  middle: {
    width: 640,
    height: 480
  },
  large: {
    width: 1280,
    height: 720
  }
}

import { mapActions } from 'vuex'

export default {
  name: 'Face',

  props: {
    type: { // 人脸对比：facecompare  人证核身：facevalidate
      validator: function (value) {
        return ['facecompare', 'facevalidate'].includes(value)
      },
      default: 'facevalidate'
    },
    size: { // small middle large
      validator: function (value) {
        return ['small', 'middle', 'large'].includes(value)
      },
      default: 'middle'
    },
    name: {
      type: String,
      default: '大牛'
    },
    cardNo: {
      type: String,
      default: ''
    }
  },

  data () {
    return {
      video_show: false,
      canvas_show: false,
      imgSrc: '',
      mediaRecorder: null,
      canvasData: null,
      isClick: false
    }
  },

  computed: {
    width () {
      return sizeType[this.size].width
    },
    height () {
      return sizeType[this.size].height
    },
    submitUrl () {
      const types = {
        facecompare: this.$url.xFaceBioDetectAndxfaceCompare,
        facevalidate: this.$url.xFaceBioDetectAndIdComparison
      }
      return types[this.type]
    }
  },

  mounted () {
  },

  beforeDestroy () {
    this.closeCamera()
  },

  methods: {
    ...mapActions([
      'setLived'
    ]),
    toValidate () { // 父组件主动验证方法
      this.toServer(this.imgSrc)
    },
    camera_options () {
      let constraints = {
        audio: true,
        video: {
          width: this.width,
          height: this.height
        }
      }
      this.getUserMedia(constraints, this.mediaSuccess, this.mediaError)
    },
    /**
       * 将图片转为file格式
       * @param {Object} dataurl 将拿到的base64的数据当做参数传递
       */
    dataURLtoBlob (dataurl) {
      let arr = dataurl.split(',')
      let mime = arr[0].match(/:(.*?);/)[1]
      let bstr = atob(arr[1])
      let n = bstr.length
      let u8arr = new Uint8Array(n)
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n)
      }
      return new Blob([u8arr], {
        type: mime
      })
    },
    /**
       *
       * @param {Object} theBlob  文件
       * @param {Object} fileName 文件名字
       */
    blobToFile (theBlob, fileName) {
      theBlob.lastModifiedDate = new Date()
      theBlob.name = fileName
      return theBlob
    },
    // 成功回调
    mediaSuccess (MediaStream) {
      console.info(MediaStream)
      let video = this.$refs.video
      video.srcObject = MediaStream
      video.play()
      this.getVideo(MediaStream)
    },
    // 失败回调
    mediaError (error) {
      console.log(error)
    },
    // 准备拍摄
    startPhoto () {
      this.video_show = true
      this.isClick = true
      // 初始化相机
      this.camera_options()
    },
    // 拍摄
    takePhoto () {
      const canvas = this.$refs.canvas
      const video = this.$refs.video
      canvas.getContext('2d').drawImage(video, 0, 0, this.width, this.height)
      this.imgSrc = canvas.toDataURL('image/png')
      this.video_show = false
      // this.toServer(this.imgSrc) // 拍照后验证
    },
    // 发送图片到服务器
    toServer (dataUrl) {
      if (!dataUrl) {
        this.$Message.warning('拍个照先～')
        return false
      }
      let blob = this.dataURLtoBlob(dataUrl)
      let file = this.blobToFile(blob, 'imgName')
      if (blob) {
        let formData = new FormData()
        formData.append('file', file)
        formData.append('authPersonName', this.name)
        formData.append('authPersonId', this.cardNo)
        this.$request.post(this.submitUrl, formData).then(res => {
          if (this.$isResSuccess(res)) {
            this.setLived('1')
            this.$Message.success('检测通过')
          }
          this.$emit('validateSuc')
        })
      }
    },
    getUserMedia (constraints, success, error) {
      if (navigator.mediaDevices.getUserMedia) {
        navigator.mediaDevices.getUserMedia(constraints).then(success).catch(error)
      } else if (navigator.webkitGetUserMedia) {
        navigator.webkitGetUserMedia(constraints, success, error)
      } else if (navigator.mozGetUserMedia) {
        navigator.mozGetUserMedia(constraints, success, error)
      } else if (navigator.getUserMedia) {
        navigator.getUserMedia(constraints, success, error)
      }
    },
    getVideo (mediaStream) {
      //  录像api的调用
      this.mediaRecorder = new MediaRecorder(mediaStream, {
        mimeType: 'video/webm'
      })
      //  停止录像以后的回调函数
      this.mediaRecorder.ondataavailable = blob => {
        //  停止以后调用上传
        console.log('-----stop', blob)
        this.uploadToServer(blob)
      }
    },
    startVideo () {
      this.mediaRecorder.start()
    },
    stopVideo () {
      this.mediaRecorder.stop()
    },
    // 发送视频到服务器
    uploadToServer (blob) {
      let file = new File([blob], 'msr-' + (new Date()).toISOString().replace(/:|\./g, '-') + '.webm', {
        type: 'video/webm'
      })
      // create FormData
      let formData = new FormData()
      formData.append('video-filename', file.name)
      formData.append('video-blob', file)
      console.log(formData)
      this.$axios.post(this.submitUrl, formData).then(res => {
        this.$emit('uploadServer', res)
      })
    },
    closeCamera () { // 关闭摄像头
      this.video_show = false
      if (!this.$refs['video'] || !this.$refs['video'].srcObject) {
        return false
      }
      let stream = this.$refs['video'].srcObject
      let tracks = stream.getTracks()
      tracks.forEach(track => {
        track.stop()
      })
      this.$refs['video'].srcObject = null
    }
  }
}
</script>

<style lang="scss" scoped>

.videos {
  display: flex;

  .left {
    width: 640px;
    height: 480px;

    .video-before {
      height: 480px;
      display: flex;
      flex-direction: column;
      align-items: center;
      background: #f8fbff;
      box-shadow: 4px 4px 0 0 rgba(222, 226, 238, .21);

      .tip {
        margin-top: 50px;
        display: flex;
        flex-direction: column;
        align-items: center;

        .type-tip {
          margin-top: 14px;
          display: flex;

          div {
            display: flex;
            align-items: center;

            &:not(:first-child) {
              margin-left: 20px;
            }

            span {
              margin-left: 8px;
            }
          }
        }
      }

      .before-img {
        width: 180px;
        height: 180px;
        margin: 40px 0;
      }

      .btn-tip {
        margin-top: 8px;
        font-size: 12px;
        color: #a5adbd;

        span {
          color: #007aff;
        }
      }
    }

    .video-after {
      display: flex;
      flex-direction: column;
      align-items: center;
      height: 100%;

      .video {
        width: 100%;
        height: 100%;
        object-fit: fill;
      }

      .btn-wrap {
        width: 100%;
        height: 60px;
        background: rgb(245, 246, 247);
        text-align: center;
        margin-top: -60px;
        z-index: 9;
        display: flex;
        align-items: center;
        justify-content: center;

        button {
          background: #dd5348;
          color: #ffffff;
          box-shadow: none;

          &:hover {
            border: none;
          }
        }
      }
    }
  }

  .right {
    display: flex;
    flex-direction: column;
    padding: 20px;
    background: #ffffff;
    border: 1px solid #dee2ee;
    box-shadow: 4px 4px 0 0 rgba(222, 226, 238, .21);

    .result-title {
      font-size: 12px;
      font-weight: 600;
      line-height: 26px;
    }

    .result-img {
      width: 240px;
      height: 180px;
      background: #f7f9fa;
      border: 1px solid #dee2ee;
    }

    .result-info {
      margin-top: 48px;
    }
  }
}

</style>
