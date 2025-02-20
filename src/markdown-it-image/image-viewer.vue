<template>
  <transition name="viewer-fade">
    <div class="el-image-viewer__wrapper" :style="{ 'z-index': zIndex }">
      <div class="el-image-viewer__mask"></div>
      <!-- CLOSE -->
      <span class="el-image-viewer__btn el-image-viewer__close" @click="hide">
        <i class="el-icon-circle-close" />
      </span>
      <!-- ARROW -->
      <template v-if="!isSingle">
        <span
          class="el-image-viewer__btn el-image-viewer__prev"
          :class="{ 'is-disabled': !infinite && isFirst }"
          @click="prev"
        >
          <i class="el-icon-arrow-left" />
        </span>
        <span
          class="el-image-viewer__btn el-image-viewer__next"
          :class="{ 'is-disabled': !infinite && isLast }"
          @click="next"
        >
          <i class="el-icon-arrow-right" />
        </span>
      </template>
      <!-- ACTIONS -->
      <div class="el-image-viewer__btn el-image-viewer__actions">
        <div class="el-image-viewer__actions__inner">
          <i class="el-icon-zoom-out" @click="handleActions('zoomOut')" />
          <i class="el-icon-zoom-in" @click="handleActions('zoomIn')" />
          <i class="el-image-viewer__actions__divider" />
          <i :class="mode.icon" @click="toggleMode" />
          <i class="el-image-viewer__actions__divider" />
          <i
            class="el-icon-refresh-left"
            @click="handleActions('anticlocelise')"
          />
          <i
            class="el-icon-refresh-right"
            @click="handleActions('clocelise')"
          />
        </div>
      </div>
      <!-- CANVAS -->
      <div class="el-image-viewer__canvas">
        <div v-for="(url, i) in urlList" :key="url" class="image-container">
          <img
            v-if="i === index"
            ref="imgRef"
            class="el-image-viewer__img"
            :src="currentImg"
            :style="imgStyle"
            @load="handleImgLoad"
            @error="handleImgError"
            @mousedown="handleMouseDown"
          />
        </div>
      </div>
    </div>
  </transition>
</template>

<script>
import { on, off, rafThrottle, isFirefox } from '../utils'
import {defineComponent, ref, watch, toRef, toRefs, nextTick, computed, reactive} from 'vue'

const Mode = {
  CONTAIN: {
    name: 'contain',
    icon: 'el-icon-full-screen',
  },
  ORIGINAL: {
    name: 'original',
    icon: 'el-icon-c-scale-to-original',
  },
}

const mousewheelEventName = isFirefox() ? 'DOMMouseScroll' : 'mousewheel'

export default defineComponent({
  name: 'elImageViewer',
  props: {
    urlList: {
      type: Array,
      default: () => [],
    },
    onSwitch: {
      type: Function,
      default: () => {},
    },
    onClose: {
      type: Function,
      default: () => {},
    },
    index: {
      type: Number,
      default: 0,
    },
  },
  emits: ['update:index'],
  setup(props, { emit }){

    const { urlList, index, onClose, onSwitch } = toRefs(props)

    const imgRef = ref(null)
    const infinite = ref(false)
    const loading = ref(false)
    const mode = ref(Mode.CONTAIN)
    const zIndex = ref(2000)
    const data = reactive({
      transform: {
        scale: 1,
        deg: 0,
        offsetX: 0,
        offsetY: 0,
        enableTransition: false,
      }
    })
    let _keyDownHandler = null
    let _mouseWheelHandler = null
    let _dragHandler = null

    // computed
    const isSingle = computed(()=> urlList.length <= 1);
    const isFirst = computed(()=> index === 0);
    const isLast = computed(()=> index === urlList.length - 1);
    const currentImg = computed(()=> urlList[index]);
    const imgStyle = computed(()=> {
      const { scale, deg, offsetX, offsetY, enableTransition } = toRef(data,'transform')
      const style = {
        transform: `scale(${scale}) rotate(${deg}deg)`,
        transition: enableTransition ? 'transform .3s' : '',
        'margin-left': `${offsetX}px`,
        'margin-top': `${offsetY}px`,
      }
      if (this.mode === Mode.CONTAIN) {
        style.maxWidth = style.maxHeight = '100%'
      }
      return style
    })

    // methods
    const hide = () => {
      deviceSupportUninstall()
      onClose()
    }

    const deviceSupportInstall = () => {
      _keyDownHandler = rafThrottle((e) => {
        const keyCode = e.keyCode
        switch (keyCode) {
          // ESC
          case 27:
            hide()
            break
          // SPACE
          case 32:
            toggleMode()
            break
          // LEFT_ARROW
          case 37:
            prev()
            break
          // UP_ARROW
          case 38:
            handleActions('zoomIn')
            break
          // RIGHT_ARROW
          case 39:
            next()
            break
          // DOWN_ARROW
          case 40:
            handleActions('zoomOut')
            break
        }
      })

      _mouseWheelHandler = rafThrottle((e) => {
        const delta = e.wheelDelta ? e.wheelDelta : -e.detail
        if (delta > 0) {
          this.handleActions('zoomIn', {
            zoomRate: 0.015,
            enableTransition: false,
          })
        } else {
          this.handleActions('zoomOut', {
            zoomRate: 0.015,
            enableTransition: false,
          })
        }
      })
      on(document, 'keydown', this._keyDownHandler)
      on(document, mousewheelEventName, _mouseWheelHandler)
    }

    const deviceSupportUninstall = () => {
      off(document, 'keydown', _keyDownHandler)
      off(document, mousewheelEventName, _mouseWheelHandler)
      _keyDownHandler = null
      _mouseWheelHandler = null
    }

    const handleImgLoad = () => {
      loading.value = false
    }

    const handleImgError = (e) => {
      loading.value = false
      e.target.alt = '加载失败'
    }

    const handleMouseDown = (e) => {
      if (loading.value || e.button !== 0) return

      const { offsetX, offsetY } = toRef(data, 'transform')
      const startX = e.pageX
      const startY = e.pageY
      _dragHandler = rafThrottle((ev) => {
        data.transform.offsetX = offsetX + ev.pageX - startX
        data.transform.offsetY = offsetY + ev.pageY - startY
      })
      on(document, 'mousemove', _dragHandler)
      on(document, 'mouseup', () => {
        off(document, 'mousemove', _dragHandler)
      })

      e.preventDefault()
    }
    
    const reset = () => {
      data.transform = {
        scale: 1,
        deg: 0,
        offsetX: 0,
        offsetY: 0,
        enableTransition: false,
      }
    }

    const toggleMode = ()=> {
      if (loading.value) return

      const modeNames = Object.keys(Mode)
      const modeValues = Object.values(Mode)
      const handleIndex = modeValues.indexOf(mode.value)
      const nextIndex = (handleIndex + 1) % modeNames.length
      mode.value = Mode[modeNames[nextIndex]]
      this.reset()
    }

    const prev = () => {
      if (isFirst.value && !infinite.value) return
      const len = urlList.length
      const handleIndex = (index - 1 + len) % len
      emit('update:index', handleIndex)
    }


    const next = () => {
      if (isLast.value && !infinite.value) return
      const len = urlList.length
      const handleIndex = (index + 1) % len
      emit('update:index', handleIndex)
    }

    const handleActions = (action, options = {}) => {
      if (loading.value) return
      const { zoomRate, rotateDeg, enableTransition } = {
        zoomRate: 0.2,
        rotateDeg: 90,
        enableTransition: true,
        ...options,
      }
      const { scale } = toRef(data, 'transform')
      switch (action) {
        case 'zoomOut':
          if (scale > 0.2) {
            data.transform.scale = parseFloat(
              (scale - zoomRate).toFixed(3)
            )
          }
          break
        case 'zoomIn':
          data.transform.scale = parseFloat((scale + zoomRate).toFixed(3))
          break
        case 'clocelise':
           data.transform.deg += rotateDeg
          break
        case 'anticlocelise':
          data.transform.deg -= rotateDeg
          break
      }
      data.transform.enableTransition = enableTransition
    }


    // watch
    watch(()=>props.index,(val) => {
      reset()
      onSwitch(val)
    })

    watch(()=>props.currentImg,() => {
      nextTick(() => {
        const $img = imgRef.value.img[0]
        if (!$img.complete) {
          loading.value = true
        }
      })
    })

    return {
      infinite,
      mode,
      loading,
      imgRef,
      ...toRefs(data),
      zIndex,
      isSingle,
      isFirst,
      isLast,
      currentImg,
      imgStyle,
      toggleMode,
      deviceSupportInstall,
      deviceSupportUninstall,
      handleImgLoad,
      handleImgError,
      handleMouseDown,
      prev,
      next,
      handleActions,
    }
  }
})

</script>

<style scoped>
.image-container {
  width: 100%;
  height: 100%;
  display: contents;
}
.el-icon-zoom-out:before {
  content: '\e776';
}

.el-icon-zoom-in:before {
  content: '\e777';
}
.el-image-viewer__wrapper {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
}
.el-image-viewer__btn {
  position: absolute;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  opacity: 0.8;
  cursor: pointer;
  box-sizing: border-box;
  user-select: none;
}
.el-image-viewer__close {
  top: 40px;
  right: 40px;
  width: 40px;
  height: 40px;
  font-size: 40px;
}
.el-image-viewer__canvas {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}
.el-image-viewer__actions {
  left: 50%;
  bottom: 30px;
  transform: translateX(-50%);
  width: 282px;
  height: 44px;
  padding: 0 23px;
  background-color: #606266;
  border-color: #fff;
  border-radius: 22px;
}
.el-image-viewer__actions__inner {
  width: 100%;
  height: 100%;
  text-align: justify;
  cursor: default;
  font-size: 23px;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: space-around;
}
.el-image-viewer__prev {
  top: 50%;
  transform: translateY(-50%);
  width: 44px;
  height: 44px;
  font-size: 24px;
  color: #fff;
  background-color: #606266;
  border-color: #fff;
  left: 40px;
}
.el-image-viewer__next {
  top: 50%;
  transform: translateY(-50%);
  width: 44px;
  height: 44px;
  font-size: 24px;
  color: #fff;
  background-color: #606266;
  border-color: #fff;
  right: 40px;
  text-indent: 2px;
}
.el-image-viewer__mask {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  opacity: 0.5;
  background: #000;
}
.viewer-fade-enter-active {
  animation: viewer-fade-in 0.3s;
}

.viewer-fade-leave-active {
  animation: viewer-fade-out 0.3s;
}

@keyframes viewer-fade-in {
  0% {
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }
  100% {
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }
}

@keyframes viewer-fade-out {
  0% {
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }
  100% {
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }
}
.el-icon-arrow-right:before {
  content: '\e6e0';
}
.el-icon-arrow-left:before {
  content: '\e6de';
}
.el-icon-full-screen:before {
  content: '\e719';
}
.el-icon-refresh-left:before {
  content: '\e6c7';
}
.el-icon-refresh-right:before {
  content: '\e6c8';
}
.el-icon-circle-close:before {
  content: '\e78d';
}
.el-icon-c-scale-to-original:before {
  content: "\e7c6";
}
@font-face {
  font-family: 'element-icons';
  src: url('../fonts/element-icons.woff') format('woff'), /* chrome, firefox */
       url('../fonts/element-icons.ttf') format('truetype'); /* chrome, firefox, opera, Safari, Android, iOS 4.2+*/
  font-weight: normal;
  font-display: auto;
  font-style: normal;
}

[class*=' el-icon-'], [class^='el-icon-'] {
  font-family: element-icons !important;
  speak: none;
  font-style: normal;
  font-weight: 400;
  font-variant: normal;
  text-transform: none;
  line-height: 1;
  vertical-align: baseline;
  display: inline-block;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
