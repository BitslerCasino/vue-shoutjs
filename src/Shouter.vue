<template>
  <div
    class="vue-shout-group"
    :style="styles"
  >
    <velocity
      @enter="enter"
      @leave="leave"
    >
      <div
        v-for="item in active"
        :key="item.id"
        class="vue-shout-wrapper"
        :data-id="item.id"
      >
        <slot
          name="body"
          :class="[classes, item.type]"
          :item="item"
          :close="() => destroy(item)"
        >
          <!-- Default slot template -->
          <div
            :class="notifyClass(item)"
            @click="destroyIfNecessary(item)"
          >
            <div
              v-if="item.title"
              class="vue-shout-title"
              v-html="item.title"
            />
            <div
              class="vue-shout-content"
              v-html="item.text"
            />
          </div>
        </slot>
      </div>
    </velocity>
  </div>
</template>
<script>
import { eventBus } from './eventBus';
import { listToDirection, parseNumericValue } from './js/util';
import plugin from './main';
import Velocity from './components/Velocity.vue';

const STATE = {
  IDLE: 0,
  DESTROYED: 2,
};
export default {
  name: 'Shout',
  components: {
    Velocity,
  },
  props: {
    group: {
      type: String,
      default: () => plugin.defaults.group,
    },
    width: {
      type: [Number, String],
      default: () => plugin.defaults.width,
    },
    reverse: {
      type: Boolean,
      default: () => plugin.defaults.reverse,
    },
    position: {
      type: [String, Array],
      default: () => plugin.defaults.position,
    },
    classes: {
      type: String,
      default: () => plugin.defaults.classes,
    },
    animation: {
      type: Object,
      default: () => plugin.defaults.velocityAnimation,
    },
    speed: {
      type: Number,
      default: () => plugin.defaults.speed,
    },
    duration: {
      type: Number,
      default: () => plugin.defaults.duration,
    },
    delay: {
      type: Number,
      default: () => plugin.defaults.delay,
    },
    max: {
      type: Number,
      default: () => plugin.defaults.max,
    },
    ignoreDuplicates: {
      type: Boolean,
      default: () => plugin.defaults.ignoreDuplicates,
    },
    closeOnClick: {
      type: Boolean,
      default: () => plugin.defaults.closeOnClick,
    },
    sync: {
      type: Boolean,
      default: () => plugin.defaults.sync,
    },
    queue: {
      type: [String, Boolean],
      default: () => plugin.defaults.queue,
    },
  },
  data() {
    return {
      list: [],
      velocity: plugin.defaults.velocity,
    };
  },
  computed: {
    actualWidth() {
      return parseNumericValue(this.width);
    },
    styles() {
      const { x, y } = listToDirection(this.position);
      const width = this.actualWidth.value;
      const suffix = this.actualWidth.type;
      const styles = {
        width: width + suffix,
        [y]: '0px',
      };
      if (x === 'center') {
        styles.left = `calc(50% - ${width / 2}${suffix})`;
      } else {
        styles[x] = '0px';
      }
      return styles;
    },
    active() {
      return this.list.filter(v => v.state !== STATE.DESTROYED);
    },
    botToTop() {
      return this.styles.hasOwnProperty('bottom');
    },
  },
  mounted() {
    eventBus.$on('shout:show', this.shout);
    eventBus.$on('shout:destroy:id', this.destroyById);
    eventBus.$on('shout:destroy:all', this.destroyAll);
  },
  beforeDestroy() {
    eventBus.$off('shout:show', this.shout);
    eventBus.$off('shout:destroy:id', this.destroyById);
    eventBus.$off('shout:destroy:all', this.destroyAll);
  },
  methods: {
    destroyIfNecessary(item) {
      if (this.closeOnClick) {
        this.destroy(item);
      }
    },
    shout(event) {
      event.group = event.group || '';
      if (this.group !== event.group) {
        return;
      }
      if (event.clean || event.clear) {
        this.destroyAll();
        return;
      }
      const duration = typeof event.duration === 'number' ? event.duration : this.duration;
      const speed = typeof event.speed === 'number' ? event.speed : this.speed;
      const {
        title, text, type, data, id,
      } = event;
      const item = {
        id,
        title,
        text,
        type,
        state: STATE.IDLE,
        speed,
        length: duration + 2 * speed,
        data,
      };
      if (duration >= 0) {
        item.timer = setTimeout(() => {
          this.destroy(item);
        }, item.length);
      }
      const direction = this.reverse ? !this.botToTop : this.botToTop;
      let indexToDestroy = -1;
      const isDuplicate = Boolean(
        this.active.find(
          shoutItem => shoutItem.title === event.title && shoutItem.text === event.text,
        ),
      );
      const canAdd = this.ignoreDuplicates ? !isDuplicate : true;
      if (!canAdd) return;
      if (direction) {
        this.list.push(item);
        if (this.active.length > this.max) {
          indexToDestroy = 0;
        }
      } else {
        this.list.unshift(item);
        if (this.active.length > this.max) {
          indexToDestroy = this.active.length - 1;
        }
      }
      if (indexToDestroy !== -1) {
        this.destroy(this.active[indexToDestroy]);
      }
    },
    notifyClass(item) {
      return ['vue-shout-template', this.classes, item.type];
    },

    destroy(item) {
      clearTimeout(item.timer);
      item.state = STATE.DESTROYED;
    },
    destroyById(id) {
      const item = this.items.find(v => v.id === id);
      if (item) {
        this.destroy(item);
      }
    },
    destroyAll() {
      this.active.forEach(this.destroy);
    },
    getAnimation(index, el) {
      const animation = this.animation[index];
      return typeof animation === 'function'
        ? animation.call(this, el)
        : animation;
    },
    enter({ el, complete }) {
      const animation = this.getAnimation('enter', el);
      this.velocity(el, animation, {
        queue: this.queue,
        sync: this.sync,
        duration: this.speed,
        complete,
      });
    },
    leave({ el, complete }) {
      const animation = this.getAnimation('leave', el);
      this.velocity(el, animation, {
        queue: this.queue,
        sync: this.sync,
        duration: this.speed,
        complete,
      });
    },
  },
};
</script>
<style>
.vue-shout-group {
  display: block;
  position: fixed;
  z-index: 5000;
}
.vue-shout-wrapper {
  display: block;
  overflow: hidden;
  width: 100%;
  margin: 0;
  padding: 0;
}
.vue-shout-title {
  font-weight: 600;
}
.vue-shout-template {
  display: block;
  box-sizing: border-box;
  background: white;
  text-align: left;
}
.vue-shout {
  display: block;
  box-sizing: border-box;
  text-align: left;
  font-size: 12px;
  padding: 10px;
  margin: 0 5px 5px;
  color: white;
  background: #44A4FC;
  border-left: 5px solid #187FE7;
}
.vue-shout.warn {
  background: #ffb648;
  border-left-color: #f48a06;
}
.vue-shout.error {
  background: #E54D42;
  border-left-color: #B82E24;
}
.vue-shout.success {
  background: #68CD86;
  border-left-color: #42A85F;
}

</style>
