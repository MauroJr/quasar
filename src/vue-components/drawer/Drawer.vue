<template>
  <div class="drawer" :class="{'left-side': !rightSide, 'right-side': rightSide}">
    <div
      class="drawer-opener touch-only mobile-only"
      v-touch-pan.horizontal="__openByTouch"
      :class="{'fixed-left': !rightSide, 'fixed-right': rightSide}"
    >&nbsp</div>
    <div
      ref="backdrop"
      class="drawer-backdrop fullscreen"
      style="background: rgba(0, 0, 0, 0.01)"
      @click="setState(false)"
      v-touch-pan.horizontal="__closeByTouch"
    ></div>
    <div
      ref="content"
      v-touch-pan.horizontal="__closeByTouch"
      class="drawer-content"
      :class="{'left-side': !rightSide, 'right-side': rightSide}"
    >
      <slot></slot>
    </div>
  </div>
</template>

<script>
import Utils from '../../utils'
import Events from '../../events'
import * as theme from '../../theme'
import Platform from '../../platform'

const
  drawerAnimationSpeed = 150,
  backdropOpacity = {
    mat: 0.7,
    ios: 0.2
  }

function getCurrentPosition (node) {
  // console.log('node', node)
  let transform = Utils.dom.style(node, 'transform')
  return transform && transform !== 'none' ? parseInt(transform.split(/[()]/)[1].split(', ')[4], 10) : 0
}

function getBetween (value, min, max) {
  if (value < min) {
    return min
  }

  if (value > max) {
    return max
  }

  return value
}

export default {
  props: {
    'right-side': Boolean,
    'swipe-only': Boolean
  },
  data () {
    return {
      opened: false
    }
  },
  methods: {
    __matToggleAnimate (percentage, done) {
      console.log('toggle', this.$refs, this.$refs.content)
      const
        node = this.$refs.content,
        backdrop = this.$refs.backdrop,
        currentPosition = getCurrentPosition(node),
        closePosition = (this.rightSide ? 1 : -1) * this.width,
        animationNeeded = this.opened || (!this.opened && percentage !== 0),
        complete = () => {
          if (!this.opened) {
            backdrop.classList.remove('active')
          }
          else {
            window.addEventListener('resize', this.close)
          }
          if (typeof done === 'function') {
            done()
          }
        }

      Velocity(node, 'stop')
      Velocity(backdrop, 'stop')

      if (this.opened) {
        backdrop.classList.add('active')
        if (!Platform.within.iframe) {
          if (!window.history.state) {
            window.history.replaceState({__quasar_drawer: true}, '')
          }
          else {
            window.history.state.__quasar_drawer = true
          }
          let state = window.history.state || {}
          state.__quasar_drawer = true
          window.history.replaceState(state, '')
          console.log('altered state', window.history.state, window.history.length)
          window.history.pushState({}, '')
          console.log('pushing state', window.history.length)
          window.addEventListener('popstate', this.__popState)
        }
      }
      else {
        window.removeEventListener('resize', this.close)
        if (!Platform.within.iframe) {
          window.removeEventListener('popstate', this.__popState)
          console.log('removed popstate')
          if (window.history.state && !window.history.state.__quasar_drawer) {
            console.log('going back')
            window.history.go(-1)
          }
        }
      }

      if (!animationNeeded) {
        complete()
        return
      }

      Velocity(
        node,
        {translateX: this.opened ? [0, currentPosition] : [closePosition, currentPosition]},
        {duration: !this.opened || currentPosition !== 0 ? drawerAnimationSpeed : 0}
      )
      Velocity(
        backdrop,
        {
          'backgroundColor': '#000',
          'backgroundColorAlpha': this.opened ? backdropOpacity.mat : 0.01
        },
        {
          duration: drawerAnimationSpeed,
          complete
        }
      )
    },
    __iosToggleAnimate (percentage, done) {
      const backdrop = this.$refs.backdrop

      if (this.opened) {
        backdrop.classList.add('active')
        document.body.classList.add('drawer-opened')
        if (!Platform.within.iframe) {
          if (!window.history.state) {
            window.history.replaceState({__quasar_drawer: true}, '')
          }
          else {
            window.history.state.__quasar_drawer = true
          }
          window.history.pushState({}, '')
          window.addEventListener('popstate', this.__popState)
        }
      }
      else {
        window.removeEventListener('resize', this.close)
        if (!Platform.within.iframe) {
          window.removeEventListener('popstate', this.__popState)
          if (window.history.state && !window.history.state.__quasar_drawer) {
            window.history.go(-1)
          }
        }
      }

      let
        currentPosition = getCurrentPosition(this.layoutContainer),
        openPosition = (this.rightSide ? -1 : 1) * this.width,
        animationNeeded = this.opened || (!this.opened && percentage !== 0),
        complete = () => {
          if (!this.opened) {
            backdrop.classList.remove('active')
            document.body.classList.remove('drawer-opened')
          }
          else {
            window.addEventListener('resize', this.close)
          }
          if (typeof done === 'function') {
            done()
          }
        }

      Velocity(this.layoutContainer, 'stop')
      Velocity(backdrop, 'stop')

      if (!animationNeeded) {
        complete()
        return
      }

      Velocity(this.layoutContainer,
        {translateX: this.opened ? [openPosition, currentPosition] : [0, currentPosition]},
        {duration: !this.opened || currentPosition !== openPosition ? drawerAnimationSpeed : 0}
      )
      Velocity(
        backdrop,
        {
          'backgroundColor': '#000',
          'backgroundColorAlpha': this.opened ? backdropOpacity.ios : 0.01
        },
        {
          duration: drawerAnimationSpeed,
          complete
        }
      )
    },
    __openByTouch (event) {
      const
        content = this.$refs.content,
        backdrop = this.$refs.backdrop

      if (Utils.dom.style(content, 'position') !== 'fixed') {
        return
      }

      let
        position = event.distance.x,
        target,
        fn,
        percentage

      if (event.isFinal) {
        this.opened = position > 75
      }

      if (theme.current === 'ios') {
        position = Math.min(position, this.width)
        percentage = 1.0 - (this.width - Math.abs(position)) / this.width
        fn = this.__iosToggleAnimate
        target = this.layoutContainer
        position = (this.rightSide ? -1 : 1) * position
      }
      else { // mat
        position = this.rightSide ? Math.max(this.width - position, 0) : Math.min(0, position - this.width)
        percentage = (this.width - Math.abs(position)) / this.width
        fn = this.__matToggleAnimate
        target = content
      }

      if (event.isFinal) {
        fn(percentage, null)
        return
      }

      target.style.transform = 'translateX(' + position + 'px)'
      backdrop.classList.add('active')
      backdrop.style.background = 'rgba(0,0,0,' + percentage * backdropOpacity[theme.current] + ')'
    },
    __closeByTouch (event) {
      const
        content = this.$refs.content,
        backdrop = this.$refs.backdrop

      let
        target, fn, percentage, position,
        initialPosition

      if (Utils.dom.style(content, 'position') !== 'fixed') {
        return
      }

      position = this.rightSide ? getBetween((event.direction === 'left' ? -1 : 1) * event.distance.x, 0, this.width) : getBetween((event.direction === 'left' ? -1 : 1) * event.distance.x, -this.width, 0)
      initialPosition = (this.rightSide ? -1 : 1) * this.width

      if (event.isFinal) {
        this.opened = Math.abs(position) <= 75
      }

      if (theme.current === 'ios') {
        position = initialPosition + position
        percentage = (this.rightSide ? -1 : 1) * position / this.width
        fn = this.__iosToggleAnimate
        target = this.layoutContainer
      }
      else { // mat
        percentage = 1 + (this.rightSide ? -1 : 1) * position / this.width
        fn = this.__matToggleAnimate
        target = content
      }

      if (event.isFinal) {
        fn(percentage, null)
        return
      }

      target.style.transform = 'translateX(' + position + 'px)'
      backdrop.style.background = 'rgba(0,0,0,' + percentage * backdropOpacity[theme.current] + ')'
    },
    setState (state, done) {
      if (
        (!this.swipeOnly && Utils.dom.viewport().width > 600) ||
        (typeof state === 'boolean' && this.opened === state)
      ) {
        if (typeof done === 'function') {
          done()
        }
        return
      }

      this.opened = !this.opened
      let fn = theme.current === 'ios' ? this.__iosToggleAnimate : this.__matToggleAnimate

      fn(this.opened ? 0.01 : 1, done)
    },
    __popState () {
      console.log('popstate')
      if (!Platform.within.iframe) {
        console.log(window.history.state)
        if (window.history.state && window.history.state.__quasar_drawer) {
          console.log('closing')
          this.setState(false)
        }
      }
    },
    open (done) {
      this.setState(true, done)
    },
    close (done) {
      this.setState(false, done)
    },
    toggle (done) {
      this.setState(!this.opened, done)
    }
  },
  mounted () {
    this.$nextTick(() => {
      const content = this.$refs.content

      if (theme.current === 'ios') {
        this.layoutContainer = this.$el.closest('.layout') || document.getElementById('quasar-app')
      }

      this.width = Utils.dom.width(content)

      ;[].slice.call(content.getElementsByClassName('drawer-closer')).forEach(el => {
        el.addEventListener('click', (event) => {
          event.stopPropagating()
          console.log('router-link closing drawer')
          this.setState(false)
        })
      })

      if (this.swipeOnly) {
        this.$el.classList.add('swipe-only')
      }

      this.__eventHandler = handler => {
        this.close(handler)
      }
      Events.$on('app::close-drawers', this.__eventHandler)
    })
  },
  beforeDestroy () {
    Events.$off('app::close-drawers', this.__eventHandler)
    this.setState(false)
  }
}
</script>
