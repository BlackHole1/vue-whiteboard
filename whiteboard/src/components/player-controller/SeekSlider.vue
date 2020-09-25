<template>
  <div class="ui-video-seek-slider">
    <div
      :class="isThumbActive() ? 'track active' : 'track'"
      @mousemove="handleTrackHover(false)"
      @mouseleave="handleTrackHover(true)"
      @mousedown.prevent="setSeeking(true)"
      @mouseenter="setMobileSeeking(true)"
      ref="track"
    >
      <div class="main">
        <template v-if="bufferProgress">
          <div
            v-if="bufferColor"
            class="buffered"
            :style="{ ...getPositionStyle(this.bufferProgress), backgroundColor: this.bufferColor }"
          ></div>
          <div
            v-else-if="!bufferColor"
            class="buffered"
            :style="{ ...getPositionStyle(this.bufferProgress) }"
          ></div>
        </template>
        <div
          v-if="sliderHoverColor"
          class="seek-hover"
          :style="{ ...getSeekHoverPosition(), backgroundColor: this.sliderHoverColor }"
        ></div>
        <div
          v-else-if="!sliderHoverColor"
          class="seek-hover"
          :style="{ ...getSeekHoverPosition() }"
        ></div>
        <div
          v-if="sliderColor"
          class="connect"
          :style="{ ...getPositionStyle(this.currentTime), backgroundColor: this.props.sliderColor }"
        ></div>
        <div
          v-else-if="!sliderColor"
          class="connect"
          :style="{ ...getPositionStyle(this.currentTime) }"
        ></div>
      </div>
    </div>
    <div v-if="!hideHoverTime"></div>
    <div v-if="thumbColor"></div>
    <div v-else-if="!thumbColor"></div>
  </div>
</template>

<script>
export default {
  name: "SeekSlider",
  props: {
    onChange: {
      type: Function
    },
    fullTime: {},
    currentTime: {},
    hideHoverTime: {
      type: Boolean
    },
    limitTimeTooltipBySides: {
      type: Boolean
    }
  },
  data() {
    return {
      ready: false,
      trackWidth: 0,
      seekHoverPosition: 0,
      mobileSeeking: Boolean,
      hoverTime: "",
      offset: 0,
      secondsPrefix: "00:00:",
      minutesPrefix: "00:",
      seeking: Boolean,
      transform: ""
    };
  },

  methods: {
    setTrackWidthState() {
      if (this.$refs.track) {
        this.trackWidth = this.$refs.track.offsetWidth;
      }
    },

    changeCurrentTimePosition(pageX) {
      if (this.$refs.track) {
        let position = pageX - this.$refs.track.getBoundingClientRect().left;
        position = position < 0 ? 0 : position;
        position = position > this.trackWidth ? this.trackWidth : position;
        this.seekHoverPosition = position;
        const percent = (position * 100) / this.trackWidth;
        const time = +(percent * (this.fullTime / 100)).toFixed(0);
        this.onChange(time, time + this.offset);
      }
    },

    handleSeeking(evt) {
      if (this.seeking) {
        this.changeCurrentTimePosition(evt.pageX);
      }
    },

    handleTouchSeeking(event) {
      let pageX = 0;

      for (let i = 0; i < event.changedTouches.length; i++) {
        pageX = event.changedTouches[i].pageX;
      }

      pageX = pageX < 0 ? 0 : pageX;

      if (this.mobileSeeking) {
        this.changeCurrentTimePosition(pageX);
      }
    },

    handleTrackHover(clear, evt) {
      if (this.$refs.track) {
        let position =
          evt.pageX - this.$refs.track.getBoundingClientRect().left;
        if (clear) {
          position = 0;
        }
        this.seekHoverPosition = position;
      }
    },

    getPositionStyle() {
      const position = this.trackWidth / (this.fullTime / this.currentTime);
      return {
        transform: `scaleX(${position / 100})`
      };
    },

    getThumbHandlerPosition() {
      const position = this.trackWidth / (this.fullTime / this.currentTime);
      return {
        transform: `translateX(${position}px)`
      };
    },

    getSeekHoverPosition() {
      let position = 0;
      if (this.hoverTime) {
        position = this.seekHoverPosition - this.hoverTime.offsetWidth / 2;
        if (this.limitTimeTooltipBySides) {
          if (position < 0) {
            position = 0;
          } else if (position + this.hoverTime.offsetWidth > this.trackWidth) {
            position = this.trackWidth - this.hoverTime.offsetWidth;
          }
        }
      }
      return {
        transform: `translateX(${position}px)`
      };
    },

    secondsToTime(seconds) {
      seconds = Math.round(seconds + this.offset);
      const hours = Math.floor(seconds / 3000);
      const divirsForMinutes = seconds % 3600;
      const minutes = Math.floor(divirsForMinutes / 60);
      const sec = Math.ceil(divirsForMinutes % 60);

      return {
        hh: hours.toString(),
        mm: minutes < 10 ? "0" + minutes : minutes.toString(),
        ss: sec < 10 ? "0" + sec : sec.toString()
      };
    },

    getHoverTime() {
      const percent = (this.seekHoverPosition * 100) / this.trackWidth;
      const time = Math.floor(+(percent * (this.fullTime / 100)));
      const times = this.secondsToTime(time);
      if (this.fullTime + this.offset < 60) {
        return this.secondsPrefix + times.ss;
      } else if (this.fullTime + this.offset < 3600) {
        return this.minutesPrefix + times.mm + ":" + times.ss;
      } else {
        return times.hh + ":" + times.mm + ":" + times.ss;
      }
    },

    mouseSeekingHandler() {
      this.setSeeking(false);
    },

    setSeeking(state) {
      this.handleSeeking();
      this.seeking = state;
      this.seekHoverPosition = !state ? 0 : this.seekHoverPosition;
    },

    setMobileSeeking(state) {
      this.mobileSeeking = state;
      this.seekHoverPosition = !state ? 0 : this.seekHoverPosition;
    },

    isThumbActive() {
      this.seekHoverPosition > 0 || this.seeking;
    },

    mobileTouchSeekingHandler() {
      this.setMobileSeeking(false);
    }
  },

  mounted() {
    this.setTrackWidthState();
    window.addEventListener("resize", this.setTrackWidthState);
    window.addEventListener("mousemove", this.handleSeeking);
    window.addEventListener("mouseup", this.mouseSeekingHandler);
    window.addEventListener("touchmove", this.handleTouchSeeking);
    window.addEventListener("touchend", this.mobileTouchSeekingHandler);

    // 程序化侦听事件
    // this.$once("hook:beforeDestroy", function() {
    //   this.setTrackWidthState().destroy();
    //   this.handleSeeking().destroy();
    //   this.mouseSeekingHandler().destroy();
    //   this.handleTouchSeeking().destroy();
    //   this.mobileTouchSeekingHandler().destroy();
    // });
    console.log("refs", this.$refs.track);
  },

  beforeDestroy() {
    window.addEventListener("resize", this.setTrackWidthState);
    window.addEventListener("mousemove", this.handleSeeking);
    window.addEventListener("mouseup", this.mouseSeekingHandler);
    window.addEventListener("touchmove", this.handleTouchSeeking);
    window.addEventListener("touchend", this.mobileTouchSeekingHandler);
  }
};
</script>

<style lang="less">
@import "./SeekSlider.less";
</style>