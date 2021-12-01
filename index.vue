<template>
<div id="photoWrap">
  <template v-if="!error">
    <div ref="container" id="container" >
      <el-progress type="circle" :percentage="percentage" :width="200" :format="()=>'loading...'" color="#999"></el-progress>
    </div>
    <div class="btns" v-if="!loading">
      <svg-icon  :icon-class="play" @click="togglePlay" :style="{background: isMove? 'hsla(0,0%,100%,.2)' : 'transparent'}"/>
      <svg-icon  icon-class="download" @click="download" />
      <svg-icon  :icon-class="fullscreen" @click="toggleFullscreen" :style="{background: isFull? 'hsla(0,0%,100%,.2)' : 'transparent'}" />
    </div>
  </template>
   <template v-else>
     <p>{{errMsg}}</p>
   </template>
</div>

</template>
<script>
import * as THREE from 'three'
import screenfull from 'screenfull'
var camera// 摄像机
var renderer// 渲染器
var scene// 舞台
var skyBox
var photoWrap
export default {
  name: 'panorama',
  props: {
    bigImg: {
      required: true
    }
  },
  data() {
    return {
      // bigImg: require('../../assets/images/test3.jpg'), // 全景图图片路径
      isMove: false,
      isFull: false,
      play: 'play',
      fullscreen: 'fullscreen-in',
      error: false,
      errMsg: '',
      loading: false,
      percentage: 5,
      timer: null
    }
  },
  mounted() {
    // 调用全景图函数
    this.$nextTick(() => {
      this.init()
      this.animate()
    })
  },
  methods: {
    // 全景图配置函数---------------
    init() {
      const _this = this
      this.loading = true
      this.timer = setInterval(() => {
        this.percentage += 5
        console.log(this.percentage)
        if (this.percentage >= 100) {
          this.percentage = 80
        }
      }, 100)

      var container = document.getElementById('container')
      photoWrap = document.getElementById('photoWrap')
      // 创建渲染器
      renderer = new THREE.WebGLRenderer()
      renderer.setPixelRatio(window.devicePixelRatio)
      // 设置画布的宽高
      renderer.setSize(photoWrap.offsetWidth, photoWrap.offsetHeight)

      // 创建场景
      scene = new THREE.Scene()
      // 创建相机
      camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100)// 四个参数分别是fov（相机摄像角度，可用于放大和缩小）、width/height（宽高比）、neer（近距离界限）、far（远距离界限）。
      camera.position.set(0, 0, 0.01)
      camera.lookAt(0, 0, 0)
      var material = new THREE.MeshBasicMaterial() // 材质
      var texture = new THREE.TextureLoader().load(
        this.bigImg,
        function() {
          _this.timer && clearInterval(_this.timer)
          _this.timer = null
          _this.percentage = 100

          // 判断容器中子元素的长度
          const childs = container.childNodes
          if (container.childNodes.length > 0) {
            container.removeChild(childs[0])
            container.appendChild(renderer.domElement)
          } else {
            container.appendChild(renderer.domElement)
          }
          _this.loading = false
          //   container.appendChild(renderer.domElement)
        },
        undefined,
        function(err) {
          _this.loading = false
          _this.error = true
          _this.errMsg = 'WebGL跨域图像 加载失败: ' + err.path[0].currentSrc
        }
      )

      material.map = texture
      skyBox = new THREE.Mesh(
        new THREE.SphereBufferGeometry(100, 100, 100),
        material
      )
      skyBox.rotation.set(0, 0.5 * Math.PI, 0)
      skyBox.geometry.scale(1, 1, -1)// -1, 1, 1   1, 1, -1
      // skyBox.rotation.y += 45.5
      scene.add(skyBox)

      window.addEventListener('resize', this.onWindowResize, false)
      var bMouseDown = false
      var x = -1
      var y = -1
      // 添加鼠标事件
      container.onmousedown = (event) => {
        event.preventDefault() // 取消默认事件
        x = event.clientX
        y = event.clientY
        bMouseDown = true
        this.isMove = false
      }
      container.onmouseup = (event) => {
        event.preventDefault()
        bMouseDown = false
        this.isMove = false
      }
      container.onmousemove = function (event) {
        event.preventDefault()
        if (bMouseDown) {
          skyBox.rotation.y += -0.005 * (event.clientX - x)
          skyBox.rotation.x += -0.005 * (event.clientY - y)
          if (skyBox.rotation.x > Math.PI / 2) {
            skyBox.rotation.x = Math.PI / 2
          }
          if (skyBox.rotation.x < -Math.PI / 2) {
            skyBox.rotation.x = -Math.PI / 2
          }
          x = event.clientX
          y = event.clientY
        }
      }
      container.onmousewheel = function (event) {
        event.preventDefault()
        if (event.wheelDelta !== 0) {
          camera.fov -= event.wheelDelta > 0 ? 1 : -1
          if (camera.fov > 120) { // 滚轮缩小最小到什么程度，数值越大缩小的越小
            camera.fov = 120
          } else if (camera.fov < 30) { // 滚轮放大最大到什么程度
            camera.fov = 30
          }
          camera.updateProjectionMatrix()
        }
      }
    },
    onWindowResize() {
      // 窗口缩放的时候,保证场景也跟着一起缩放
      camera.aspect = photoWrap.offsetWidth / photoWrap.offsetHeight
      camera.updateProjectionMatrix()
      renderer.setSize(photoWrap.offsetWidth, photoWrap.offsetHeight)
    },
    animate() {
      requestAnimationFrame(this.animate)
      if (this.isMove) {
        skyBox.rotation.y += Math.PI * 0.001
        // skyBox.rotation.x += Math.PI * 0.001
      }
      renderer.render(scene, camera)
    },
    // 自动旋转
    togglePlay() {
      this.isMove = !this.isMove
      this.play = this.isMove ? 'play-active' : 'play'
    },
    // 下载
    download() {
      const link = document.createElement('a')
      link.style.display = 'none'
      link.href = this.bigImg
      link.setAttribute('download', this.bigImg)
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    },
    //  全屏
    toggleFullscreen() {
      if (!screenfull.isEnabled) {
        this.$message({
          message: '你的浏览器不支持',
          type: 'warning',
          showClose: true
        })
        return false
      }
      screenfull.toggle()
      screenfull.on('change', () => {
        if (screenfull.isFullscreen) {
          // 全屏时，要执行的操作
          photoWrap.classList.add('psv-container--fullscreen')
          this.onWindowResize()
        } else {
          // 取消全屏时，要执行的操作
          photoWrap.classList.remove('psv-container--fullscreen')
          this.onWindowResize()
        }
        this.isFull = screenfull.isFullscreen
        this.fullscreen = this.isFull ? 'fullscreen-out' : 'fullscreen-in'
      })
    }
  },
  beforeDestroy() {
    this.isMove = false
    this.error = false
    this.errMsg = ''
    cancelAnimationFrame(this.animate)
    this.timer && clearInterval(this.timer)
  }
}
</script>

<style lang="scss">
.photo_wrap{
  margin-bottom:0;
 .el-drawer__body{
    height:calc(100% - 45px);
  }
  .el-dialog__body{
    height: 70vh;
  }
}
.psv-container--fullscreen {
  position: fixed!important;
  height:100vh!important;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
}
#photoWrap {
  width: 100%;
  height: 100%;
  position:relative;
  #container{
    width:100%;
    height:100%;
  }
  .btns{
    display: flex;
    position: absolute;
    z-index: 90;
    bottom: 0px;
    left: 0;
    width: 100%;
    height: 40px;
    background: rgba(61,61,61,.5);
    transition: bottom .1s ease-in-out;
    overflow: hidden;
    .svg-icon{
      width: 25px;
      height: 25px;
      padding: 7.5px;
      cursor: pointer;
      background: transparent;
      color: hsla(0,0%,100%,.7);
      &:hover {
        transform: scale(1.2);
      }
    }
  }
  .el-progress{
    width:100%;
    height:100%;
  }
  .el-progress-circle {
    position: absolute;
    top: 50%;
    left: 50%;
    margin-top: -100px;
    margin-left:-100px;
  }
}

</style>
