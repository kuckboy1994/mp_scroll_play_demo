## 演示
![](https://wx4.sinaimg.cn/mw690/ec4d7780ly1fsvyopp7qig207v0dw1l5.gif)

1. 滚动到视频一半的位置，进行切换视频播放
2. 记录所有视频的播放位置，再次切换回来的时候继续播放。
3. 当播放视频滚出可见范围的时候，立刻停止播放（声音停止）。滚动停止的时候新的视频开始播放。
4. 全屏播放。（这里存在一个坑，全屏的时候会立刻自动退出全屏）
5. 基础上拉加载，上拉刷新（demo中未提现，配置下app.json即可实现）

## 代码解读
```js
/* global getApp Page wx */
const app = getApp()

// 基础版的防抖函数
let timeout
function debounce (func, wait) {
  return function () {
    clearTimeout(timeout)
    timeout = setTimeout(func, wait)
  }
}

// 基础版的节流函数
let timeout2 = null
function throttle (func, wait) {
  return function () {
    if (!timeout2) {
      timeout2 = setTimeout(function () {
        clearTimeout(timeout2)
        timeout2 = null
        func()
      }, wait)
    }
  }
}

Page({
  data: {
    width: wx.getSystemInfoSync().windowWidth,
    height: wx.getSystemInfoSync().windowHeight,

    videoLoading: false,
    videoList: [],

    location: [],
    isLock: false,  // 当前栗子无用，如果有些弹窗控制不住背后的视频列表滚动的话，isLock的作用就发挥出来了。
    localIndex: 0,
    noPageScroll: false
  },
  info: {
    videoPlayDetail: {} // 存放所有视频的播放位置
  },
  onLoad (options) {
    this.getVideoList(1)
  },
  /**
   * 分页获取视频
   */
  getVideoList (initPage) {
    const { videoLoading, videoList } = this.data
    if (videoLoading) {
      return
    }
    this.setData({
      videoLoading: true
    })
    // 制造模拟数据
    let data = [{
      cover: 'https://wx4.sinaimg.cn/mw690/ec4d7780ly1fsvx1996xoj20da07in0t.jpg',
      src: 'http://wxsnsdy.tc.qq.com/105/20210/snsdyvideodownload?filekey=30280201010421301f0201690402534804102ca905ce620b1241b726bc41dcff44e00204012882540400&bizid=1023&hy=SH&fileparam=302c020101042530230204136ffd93020457e3c4ff02024ef202031e8d7f02030f42400204045a320a0201000400',
      width: 480,
      height: 272,
      title: '腾讯大学视频',
      isPlay: false
    }]
    data = data.concat(data)
    data = data.concat(data)
    data = data.concat(data)

    console.log(data)
    // 模拟请求
    setTimeout(() => {
      this.setData({
        videoList: videoList.concat(this.formatVideoList(data)),
        videoLoading: false
      })
      // 给数据足够的渲染时间，之后进行视频位置的测量
      setTimeout(() => {
        this.getLocationInfo()
        if (initPage) { // 如果是首次进入，就自动播放第一个视频
          this.showVideoList(0)
        }
      }, 1000)
    }, 1000)
  },
  /**
   * 格式化视频列表
   * @param {Array} videoList 需要格式化的视频列表
   */
  formatVideoList (videoList) {
    const { width, height } = this.data
    return videoList.map(value => {
      let styleHight = width * value.height / value.width
      styleHight = styleHight > 0.7 * height ? 0.7 * height : styleHight
      return {
        ...value,
        styleHight: Math.floor(styleHight),
        currentTime: 0,
        isPlay: false
      }
    })
  },
  // 点击播放
  eventPlay (event) {
    const { index } = event.currentTarget.dataset
    this.showVideoList(index)
  },
  // 视频更新的时候不断的去记录播放的位置
  eventPlayupdate (event) {
    const { detail: { currentTime }, currentTarget: { dataset: { index } } } = event

    this.info.videoPlayDetail[index] = currentTime
  },
  // 到底加载更多
  onReachBottom () {
    this.getVideoList()
  },
  // 设置播放的视频
  showVideoList (index) {
    console.log(index)
    let { videoList } = this.data
    videoList = videoList.map(value => {
      value.isPlay = false
      return value
    })
    if (index >= 0 && videoList[index]) {
      videoList[index].isPlay = true
    }
    this.setData({
      videoList
    })
  },
  // 全屏播放，设置当前页面滚动无效，全屏的时候，会触发滚动事件。
  eventFullScreen (event) {
    console.log(event)
    const { fullScreen } = event.detail
    this.setData({
      noPageScroll: fullScreen
    })
  },
  onPageScroll ({ scrollTop }) {
    if (this.data.noPageScroll) {
      return
    }
    // 获取数据分别放置到各自的节流防抖函数中，防止调用的时候数据已近发生改变

    // 节流，每200毫秒触发一次
    throttle(() => {
      console.log('throttle')
      let { location, localIndex, videoList } = this.data
      let index = 0
      for (let i = 0; i < location.length; i++) {
        if (location[i].start <= scrollTop && location[i].end >= scrollTop) {
          index = i
          break
        }
      }

      if (localIndex !== index) {
        videoList[localIndex].currentTime = this.info.videoPlayDetail[localIndex]
        this.setData({
          videoList
        })
        this.showVideoList()
      }
    }, 200)()

    // 防抖，只触发一次
    debounce(() => {
      console.log('debounce')
      let { location, isLock } = this.data
      if (!isLock) {
        let index = 0
        for (let i = 0; i < location.length; i++) {
          if (location[i].start <= scrollTop && location[i].end >= scrollTop) {
            index = i
            break
          }
        }

        this.showVideoList(index)
        this.setData({
          localIndex: index
        })
      }
    }, 200)()
  },
  // 测量当前的所有视频的高度，计算出视频播放与否的位置
  getLocationInfo () {
    const { videoList } = this.data
    var query = wx.createSelectorQuery()
    for (let i = 0; i < videoList.length; i++) {
      query.select(`#video${i}`).boundingClientRect()
    }

    const length = videoList.length
    const location = []
    query.exec(res => {
      console.log(res)

      let start = 0
      let end = 0
      let lei = 0
      for (let i = 0; i < length; i++) {
        if (i === 0) {
          start = 0
          end = videoList[i]['styleHight'] / 2
        } else {
          start = end
          end = lei + videoList[i]['styleHight'] / 2
        }

        location.push({
          start: start,
          end: end
        })
        lei = lei + res[i].height + 10
      }
      this.setData({
        location
      })
    })
  }
})
```