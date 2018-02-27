<template>
  <div class='scroller select-list extension-list'>
    <ol class='scroller-wrapper list-group' ref='wrapper' :class="{'forbid-scroll':!scrollable , 'vertical': !horizontal, 'horizontal': horizontal }">
      <li class='scroller-item' v-for="(item, index) in items" :class="{'selected': current === index, 'on-wrapper': landmarks[index] === 0}" v-on:click='itemClick(index)'>
        <slot :index='index' :item="item"></slot>
      </li>
    </ol>
  </div>
</template>

<script>
// 需要安装vueify
// https://github.com/electron-userland/electron-compile/issues/192
import { debounce } from 'lodash'

export default {
  name: 'scroller',
  data() {
    return {
      current: -1,
      offset: 0,
      wrapperLength: null,
      landmarks: [], // [-1, -1, 0, 0, 1, 1] marks
      scrollListener: null
    }
  },
  props: ["items"],
  computed: {
    horizontal(){
      return this.$attrs["horizontal"] !== undefined
    },
    scrollable(){
      return this.$attrs["scrollable"] !== undefined
    },
    vertical(){
      return !this.horizontal;
    }
  },
  watch: {
    items: function(newValue) {
      this.$nextTick(this.initScrollable)
      this.current = -1
    },
    current: function(newValue) {
      console.log('current change', newValue, this.vertical)
      let vm = this

      if (newValue < -1 || newValue >= vm.items.length) return
      if (newValue === -1) vm.$emit('current-change', newValue)

      if (vm.landmarks[newValue] < 0) {
        vm.moveTo(newValue, true, () => { vm.$emit('current-change', newValue) })
      } else if (vm.landmarks[newValue] > 0) {
        vm.moveTo(newValue, false, () => { vm.$emit('current-change', newValue) })
      } else {
        vm.$emit('current-change', newValue)
      }
    }
  },
  methods: {
    initScrollable() {
      let vm = this

      // marker wrapper length
      let wrapper = vm.$refs['wrapper']

      // reset scrollLeft
      wrapper.scrollTop = 0
      wrapper.scrollLeft = 0
      vm.offset = 0

      // calculate wrapperLength
      vm.wrapperLength = vm.vertical ? wrapper.offsetHeight : wrapper.offsetWidth
      console.log('wrapperLength', vm.wrapperLength)

      // calculate each children's position
      vm.landmark()
    },
    landmark() {
      // mark whether or not an item is in the wrapper view box
      // based on landmarks
      let vm = this
      let children = vm.$refs['wrapper'].children

      vm.landmarks = []
      for (let index = 0; index < children.length; index++) {
        let child = children[index]
        // if an item in the view, then (WRAPPER_LEFT - ITEM_LEFT) > 0 and (WRAPPER_RIGHT - ITEM_RIGHT) < 0
        // Outboundary happens when either too LEFT or too RIGHT, the multiple would always be greater than zero
        // thus, we use (WRAPPER_LEFT - ITEM_LEFT) * (WRAPPER_RIGHT * ITEM_RIGHT) > 0 to judge whether or not an item is in the wrapper current now
        let flag1 = vm.offset - (vm.vertical ? child.offsetTop : child.offsetLeft)
        let flag2 = (vm.offset + vm.wrapperLength) - (vm.vertical ? child.offsetTop + child.offsetHeight : child.offsetLeft + child.offsetWidth)

        // vm.landmarks[index] = flag1 * flag2 <= 0
        if (flag1 > 0 && flag2 > 0) {
          vm.landmarks[index] = -1
        } else if (flag1 < 0 && flag2 < 0) {
          vm.landmarks[index] = 1
        } else {
          vm.landmarks[index] = 0
        }
      }
      // console.log('update landmarks', vm.landmarks)

      vm.$emit('landmarks-change', vm.landmarks)
    },
    moveTo(index, isFront = true, cb = null) {
      let vm = this
      let target = vm.$refs['wrapper'].children[index]

      if (isFront) {
        // offset is target LEFT
        vm.offset = vm.vertical ? target.offsetTop : target.offsetLeft

        // apply offset to scroll
        vm.applyScroll(cb)
        // vm.$refs['wrapper'].scrollLeft = vm.offset
      } else {
        // offset + wrapperLength is target RIGHT
        vm.offset = (vm.vertical ? target.offsetTop + target.offsetHeight : target.offsetLeft + target.offsetWidth) - vm.wrapperLength

        // recalculate on-wrapper status
        vm.landmark()
        // 右侧对齐标记哪些Item是需要显示后，再取第一个进行左侧对齐
        vm.moveTo(vm.landmarks.indexOf(0), true, cb)
      }
    },
    applyScroll(cb) {
      // this function only handle details on how to use requireAnimationFrame to scroll wrapper
      let vm = this
      let wrapper = vm.$refs['wrapper']
      let startTime = null
      let startPosition = (vm.vertical ? wrapper.scrollTop : wrapper.scrollLeft)
      let endPosition = vm.offset
      let totalTime = 200 // 0.2 second for complete
      console.log('apply scroll', startPosition, endPosition)
      let scroll = function(timestamp) {
        if (!startTime) startTime = timestamp
        let progress = timestamp - startTime

        let currentOffset = startPosition + (endPosition - startPosition) * progress / totalTime

        // apply
        if (vm.vertical) {
          wrapper.scrollTop = currentOffset
        } else {
          wrapper.scrollLeft = currentOffset
        }

        if (progress < totalTime) {
          window.requestAnimationFrame(scroll)
        } else {
          if (typeof cb === 'function') cb()
        }
      }
      window.requestAnimationFrame(scroll)
    },
    setCurrent(index) {
      console.log('setCurrent called', index)
      if (index < 0 || index >= this.items.length) return
      this.current = index
    },
    clearCurrent() {
      this.current = -1
    },
    nextPage() {
      let index = this.landmarks.indexOf(1)

      if (index >= 0) {
        this.moveTo(index)
      }
    },
    prevPage() {
      let index = this.landmarks.lastIndexOf(-1)
      if (index >= 0) {
        this.moveTo(index, false)
      }
    },
    nextCurrent(loop = false) {
      let target = this.current + 1
      if (!loop && target > this.items.length - 1) return false;
      this.current = target % this.items.length
      return true
    },
    prevCurrent(loop = false) {
      let target = this.current - 1
      if (!loop && target < 0) return false;
      this.current = target % this.items.length
      return true;
    },
    // handle item click
    itemClick(index) {
      this.current = index
      this.$emit('item-click', index)
    }
  },
  mounted() {
    // add a listener on wrapper scroll, to update correct offset for landmark use
    let vm = this
    let wrapper = vm.$refs['wrapper']

    vm.scrollListener = debounce(function(e) {
      vm.offset = (vm.vertical ? wrapper.scrollTop : wrapper.scrollLeft)
      vm.landmark()
    }, 100)
    wrapper.addEventListener('scroll', vm.scrollListener)
  }
}
</script>

<style lang="less">
.scroller {
  display: block;
  position: relative;
  height: 100%;
  width: 100%;

  .scroller-wrapper {
    width: 100%;
    height: 100%;
    position: relative;
    display: flex;
    overflow: scroll;

    &::-webkit-scrollbar {
      display: none;
    }

    &.horizontal {
      .scroller-item {
        margin: 0 0.2em;

        &:first-of-type {
          margin-left: 0;
        }
        &:last-of-type {
          margin-right: 0;
        }
      }
    }

    &.vertical {
      flex-direction: column;
    }

    &.forbid-scroll {
      overflow: hidden;
    }
  }
}
</style>