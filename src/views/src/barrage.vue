
<template>
  <div class="z_barrage-container">
    <canvas
            :ref="utilCanvasId"
            :width="containerWidth"
            :height="containerHeight"
            style="display: none;"
    />
    <div
            class="z_container"
            :style="{height: containerHeight/2+'px'}"
    >
      <canvas
              :id="canvasId"
              :ref="canvasId"
              class="z_barrage"
              :width="containerWidth"
              :height="containerHeight"
              :style="{'width': containerWidth/2 + 'px',
                 'height': containerHeight/2 + 'px'}"
      />
    </div>
  </div>
</template>

<script>
export default {
  name: 'Barrage',
  props: {
    barrageList: {
      type: Array,
      default: () => []
    },
    speed: {
      type: Number,
      default: 4
    },
    loop: {
      type: Boolean,
      default: true
    },
    channels: {
      type: Number,
      default: 2
    },
    barrageHeight: {
      type: Number,
      default: 60
    },
    borderColor: {
      type: String,
      default: ''
    },
    background: {
      type: String,
      default: ''
    },
    linearGradient: {
      type: Object,
      default: () => {
        return {
          startColor: '',
          endColor: ''
        }
      }
    },
    fontSize: { // 文字大小
      type: Number,
      default: 30
    },
    avatarBorderColor: {
      type: String,
      default: ''
    },
    avatarBorderWidth: {
      type: Number,
      default: 0
    },
    avatarWidth: {
      type: Number,
      default: 48
    },
    horizontalGap: {
      type: Number,
      default: 100
    },
    verticalGap: {
      type: Number,
      default: 20
    },
    width: {
      type: Number,
      default: document.body.clientWidth * 2
    },
    // 文字render
    textRender: {
      type: Function,
      default(context, data, x, y, component) {
        component.drawText(context, data.content, x, y)
      },
    },
    // 头像render
    avatarRender: {
      type: Function,
      default(context, data, x, y, r, component) {
        component.drawAvatar(context, data.icon, x, y, r)
      }
    },
    // 背景render
    backgroundRender: {
      type: Function,
      default(context, data, x, y, width, height, radius, component) {
        component.drawBackground(context, data.bgColor, x, y, width, height, radius)
      }
    }
  },
  data () {
    return {
      newBarrageArray: [], // 新增弹幕之后的总弹幕
      barrageArray: [],
      barrageQueue: [],
      containerWidth: 0,
      containerHeight: 0,
      channelsArray: [],
      barrageChannels: 1,
      playing: false,
      canvasId: `barrage_cvs_${new Date().getTime()}`,
      utilCanvasId: `barrage_util_cvs_${new Date().getTime()}` // 用于进行文字测量计算的临时canvas
    }
  },
  watch: {
    barrageList: {
      handler(val) {
        this.$nextTick(() => {
          if (val.length !== 0) {
            this.barrageQueue = JSON.parse(JSON.stringify(val))
            this.newBarrageArray = JSON.parse(JSON.stringify(val))
            this.initData()
            if (!this.playing) {
              window.requestAnimationFrame(this.render)
              this.playing = true
            }
          }
        })
      },
      immediate: true
    }
  },
  mounted () {
    this.containerWidth = this.width
    this.barrageChannels = this.channels // 总高度对应的轨道数
    this.containerHeight =  this.channels * (this.barrageHeight + this.verticalGap) + this.verticalGap // 设定总高度
    this.ctx = this.$refs[this.canvasId].getContext('2d')
    this.ctx1 = this.$refs[this.utilCanvasId].getContext('2d')
    this.barrageClickEvent()
  },
  methods: {
    /**
     * 数据初始化
     */
    initData () {
      for (let i = 0; i < this.barrageQueue.length; i++) { // 此处处理只显示50个字符
        let tagImg = null
        let img = null
        if (this.barrageQueue[i].icon) {
          img = new Image()
          img.src = this.barrageQueue[i].icon
        }
        if (this.barrageQueue[i].tagImage) {
          tagImg = new Image()
          tagImg.src = this.barrageQueue[i].tagImage
        }
        let content = this.dealStr(this.barrageQueue[i].content)
        this.barrageArray.push({
          ...this.barrageQueue[i],
          content: content,
          x: this.containerWidth + (+this.horizontalGap),
          icon: img,
          tagImage: tagImg,
          width: this.ctx1.measureText(content).width * 3 + (this.barrageQueue[i].icon ? 60 : 0),
          color: this.barrageQueue[i].color || '#FFFFFF',
          bgColor: this.barrageQueue[i].bgColor || 'rgba(0,0,0,0.4)'
        })
      }
      this.initChannel()
    },
    /**
     * 初始化轨道数据
     */
    initChannel () {
      for (let i = 0; i < this.barrageChannels; i++) {
        let item = this.barrageArray.shift()
        if (item) {
          this.channelsArray[i] = [item]
        } else {
          this.channelsArray[i] = []
        }
      }
    },
    /**
     * 渲染
     */
    render () {
      this.ctx.clearRect(0, 0, this.containerWidth, this.containerHeight)
      this.ctx.font = `${this.fontSize}px Microsoft YaHei`
      this.draw()
      window.requestAnimationFrame(this.render)
    },
    draw () {
      for (let i = 0; i < this.channelsArray.length; i++) {
        for (let j = 0; j < this.channelsArray[i].length; j++) {
          try {
            let barrage = this.channelsArray[i][j]
            barrage.x -= this.speed
            if (barrage.x <= this.containerWidth) { // 弹幕显示
              const gap = this.verticalGap; // 每个轨道之间的缝隙
              const currTop = i * (this.barrageHeight + gap) + gap // 当前弹幕条的顶部
              const currMiddle = currTop + this.barrageHeight / 2 // 当前弹幕条的中间y位置
              this.borderColor && this.drawRoundRectBorder(this.ctx, barrage.x - this.barrageHeight / 2, currTop, barrage.width + this.barrageHeight, this.barrageHeight, this.barrageHeight / 2)
              this.backgroundRender(this.ctx, barrage, barrage.x - this.barrageHeight / 2, currTop + 1, barrage.width + this.barrageHeight, this.barrageHeight - 2, this.barrageHeight / 2, this)
              this.ctx.fillStyle = `${barrage.color}`
              this.textRender(this.ctx, barrage, barrage.x + (barrage.icon ? this.barrageHeight / 2 + this.fontSize : -5), currMiddle + this.fontSize / 2 - this.fontSize * 0.1, this) // 行高高于字体大小
              if (barrage.icon) {
                this.avatarRender(this.ctx, barrage, barrage.x - 10, currMiddle - this.avatarWidth / 2, this.avatarWidth / 2, this)
              }
              if (barrage.tagImage) {
                this.originImg(this.ctx, barrage.tagImage, barrage.x - this.barrageHeight - 10, currTop, this.barrageHeight, this.barrageHeight)
              }
            }
            if (barrage.x < -(barrage.width + this.barrageHeight)) { // 弹幕删除
              let arr = this.channelsArray.reduce((a, b) => a.concat(b))
              if (this.loop) {
                if (this.checkBarrageStatus(arr)) {
                  this.barrageQueue = []
                  this.barrageQueue = JSON.parse(JSON.stringify(this.newBarrageArray))
                  this.initData()
                }
              }
            }
            // 弹幕插入时机判断
            if (barrage.x <= Math.floor((this.containerWidth - barrage.width - 40)) && barrage.x >= Math.floor(this.containerWidth - barrage.width - 40 - this.speed) && (j === this.channelsArray[i].length - 1) && this.barrageArray.length !== 0) {
              let item = this.barrageArray.shift()
              this.channelsArray[i].push(item)
            }
          } catch (e) {
            console.log(e)
          }
        }
      }
    },
    /**
     * 添加数据
     */
    add (obj) {
      let content = this.dealStr(obj.content)
      let img = null
      let tagImg = null
      if (obj.icon) {
        img = new Image()
        img.src = obj.icon
      }
      if (obj.tagImage) {
        tagImg = new Image()
        tagImg.src = obj.tagImage
      }
      let item = {
        ...obj,
        content: content,
        x: this.containerWidth + this.barrageHeight,
        icon: obj.icon ? img : '',
        tagImage: obj.tagImage ? tagImg : '',
        width: this.ctx1.measureText(content).width * 3 + (obj.icon ? this.barrageHeight : 0),
        color: obj.color || '#FFFFFF',
        bgColor: obj.bgColor || 'rgba(0,0,0,0.4)'
      }
      let originItem = {
        ...obj,
        color: obj.color || '#FFFFFF',
        bgColor: obj.bgColor || 'rgba(0,0,0,0.4)'
      }
      if (this.barrageArray.length === 0) { // 剩余弹幕数为0
        this.newBarrageArray.unshift(originItem)
      } else {
        this.barrageArray.unshift(item)
        let insertIndex = this.barrageList.length - this.barrageArray.length
        this.newBarrageArray.splice(insertIndex, 0, originItem)
      }
    },
    /**
     * 弹幕点击事件
     */
    barrageClickEvent () {
      document.getElementById(this.canvasId).addEventListener('click', (e) => {
        const p = this.getEventPosition(e)
        let channelIndex = Math.floor(p.y / (this.barrageHeight + 30))
        if(channelIndex > this.channels - 1) {
          return;
        }
        const channelsArray = this.channelsArray[channelIndex]
        for (let i = 0; i < channelsArray.length; i++) {
          let channelItemArray = Object.assign({}, channelsArray[i])
          if (p.x > channelItemArray.x && p.x < (channelItemArray.x + channelItemArray.width)) {
            // 该条弹幕点击
            this.$emit('click-item', channelItemArray)
            if (channelItemArray.icon && p.x < channelItemArray.x + this.barrageHeight / 2 + 20) {
              // 弹幕头像点击
              this.$emit('click-icon', channelItemArray)
            } else {
              // 弹幕内容区点击
              this.$emit('click-content', channelItemArray)
            }
            break;
          }
        }
      }, false)
    },
    /**
     * 获取点击位置
     */
    getEventPosition (ev) {
      let x, y
      if (ev.layerX || ev.layerX === 0) {
        x = ev.layerX
        y = ev.layerY
      } else if (ev.offsetX || ev.offsetX === 0) {
        x = ev.offsetX
        y = ev.offsetY
      }
      return { x: 2 * x, y: 2 * y }
    },
    /**
     * 判断所有的弹幕是否滚动完成
     * @params arr
     */
    checkBarrageStatus (arr) {
      for (let i = 0; i < arr.length; i++) {
        if (arr[i].x > -arr[i].width) return false
      }
      return true
    },
    /**
     * 处理字符
     */
    dealStr (str) {
      return str.length > 50 ? `${str.substring(0, 50)}...` : str
    },
    /**
     * 获取随机颜色
     */
    getColor () {
      return `#${Math.floor(Math.random() * 16777215).toString(16)}`
    },
    drawText(context, text, x, y) {
      context.fillText(text, x, y)
    },
    /**
     * 裁剪图片
     * @param ctx
     * @param img
     * @param x
     * @param y
     * @param r
     */
    drawAvatar (ctx, img, x, y, r) {
      ctx.save()
      let d = 2 * r
      let cx = x + r
      let cy = y + r
      ctx.beginPath()
      ctx.arc(cx, cy, r, 0, 2 * Math.PI)
      ctx.clip()
      try {
        ctx.drawImage(img, x, y, d, d)
      } catch (e) {
        console.warn(`[barrage],drawAvatar,头像绘制失败, icon: ${img.src}`)
      }
      if (this.avatarBorderColor || this.avatarBorderWidth) {
        ctx.strokeStyle = this.avatarBorderColor
        ctx.lineWidth = this.avatarBorderWidth
        ctx.stroke()
      }
      ctx.restore()
      ctx.closePath()
    },
    /**
     * 绘制原始图片
     * @param ctx
     * @param img
     * @param x
     * @param y
     * @param width
     * @param height
     */
    originImg (ctx, img, x, y, width, height) {
      try {
        ctx.beginPath()
        ctx.drawImage(img, x, y, width, height)
        ctx.closePath()
      } catch (e) {
        console.log(e)
      }
    },
    /**
     * 绘画圆角矩形
     * @param context
     * @param bgColor
     * @param x
     * @param y
     * @param width
     * @param height
     * @param radius
     */
    drawBackground (context, bgColor, x, y, width, height, radius) {
      if (this.linearGradient.startColor && this.linearGradient.endColor) {
        let linearGrad = context.createLinearGradient(x, y, x, y + height)
        linearGrad.addColorStop(0, this.linearGradient.startColor)
        linearGrad.addColorStop(1, this.linearGradient.endColor)
        context.fillStyle = linearGrad || bgColor
      } else {
        context.fillStyle = this.background || bgColor
      }
      context.beginPath()
      context.arc(x + radius, y + radius, radius, Math.PI, Math.PI * 3 / 2)
      context.lineTo(width - radius + x, y)
      context.arc(width - radius + x, radius + y, radius, Math.PI * 3 / 2, Math.PI * 2)
      context.lineTo(width + x, height + y - radius)
      context.arc(width - radius + x, height - radius + y, radius, 0, Math.PI / 2)
      context.lineTo(radius + x, height + y)
      context.arc(radius + x, height - radius + y, radius, Math.PI / 2, Math.PI)
      context.fill()
      context.closePath()
    },
    /**
     * 绘画圆角矩形
     * @param context
     * @param x
     * @param y
     * @param width
     * @param height
     * @param radius 半径
     */
    drawRoundRectBorder (context, x, y, width, height, radius) {
      context.beginPath()
      context.lineWidth = 2
      context.strokeStyle = this.borderColor
      context.arc(x + radius, y + radius, radius, Math.PI, Math.PI * 3 / 2)
      context.lineTo(width - radius + x, y)
      context.arc(width - radius + x, radius + y, radius, Math.PI * 3 / 2, Math.PI * 2)
      context.lineTo(width + x, height + y - radius)
      context.arc(width - radius + x, height - radius + y, radius, 0, Math.PI / 2)
      context.lineTo(radius + x, height + y)
      context.arc(radius + x, height - radius + y, radius, Math.PI / 2, Math.PI)
      context.stroke()
      context.closePath()
    }
  }
}
</script>

<style lang="css" scoped>
.z_container {
  width: 100%;
  overflow: hidden;
}

</style>
