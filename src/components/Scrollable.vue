<template>
  <div class="v3-scrollable">
    <div
      ref="hover1Ref"
      class="v3-scrollable__hover1"
      @mouseenter="onMouseEntered"
    />
    <div
      ref="hover2Ref"
      class="v3-scrollable__hover2"
      @mouseenter="onMouseEntered"
    />
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";


export const scrollEle = (ele: HTMLElement, vel = 10): void => {
  ele.scrollLeft += vel;
};

export const getHorizontalHoverScrollDirection = (
  e: MouseEvent,
  ele: HTMLElement,
  areaWidth: number
): 1 | -1 | 0 => {
  // console.log(e);
  const rect = ele.getBoundingClientRect();
  const elementWidth = rect.width;
  const relativeX = e.pageX - rect.left - document.body.scrollLeft;
  const scrollLeft = ele.scrollLeft;
  const scrollWidth = ele.scrollWidth;
  const offset = ele.offsetWidth;
  // console.log(
  //   `scrollLeft ${scrollLeft}, scrollWidth ${scrollWidth}, offset ${offset}`
  // );
  if (elementWidth - relativeX < areaWidth) {
    if (scrollWidth - scrollLeft === offset) return 0;
    return 1;
  } else if (relativeX < areaWidth && scrollLeft) {
    return -1;
  }
  return 0;
};

export class Scrollable {
  container: HTMLElement;
  areaWidth: number;
  canScrollOnHover: boolean;
  canScrollOnWheel: boolean;
  velocity: number;
  _handle = -1;
  _inertia = 0;
  _leftHoverEl: HTMLElement;
  _rightHoverEl: HTMLElement;
  _offset: number;
  _backgroundColor: string;

  constructor(
    ele: HTMLElement,
    backgroundColor = '123, 123, 123',
    canScrollOnHover = false,
    canScrollOnWheel = true,
    velocity = 10,
    areaWidth = 70
  ) {
    if (!ele) {
      throw new Error("Invalid HTMLElement");
    }
    this.container = ele;
    this.canScrollOnHover = canScrollOnHover;
    this.canScrollOnWheel = canScrollOnWheel;
    this.velocity = velocity;
    this.areaWidth = areaWidth;
    this._offset = 0;
    this._backgroundColor = backgroundColor;

    this._leftHoverEl = document.createElement("div");
    this._rightHoverEl = document.createElement("div");
  
    this._leftHoverEl.className = "leftHover";
    this._leftHoverEl.classList.add("hidden");
    this._rightHoverEl.className = "rightHover";

    this.container.classList.add("scrollable");
    this.container.style.setProperty('--color', this._backgroundColor);

    this._leftHoverEl.addEventListener("mouseover", () => {
      // console.log("mouseover left")
      this.scrollOnHover(-1);
    });
    this._leftHoverEl.addEventListener("mouseleave", () => {
      // console.log("mouseleave left")
      this.stop();
    });
    this._rightHoverEl.addEventListener("mouseover", () => {
      this.scrollOnHover(1);
    });
    this._rightHoverEl.addEventListener("mouseleave", () => {
      this.stop();
    });
    this.container.appendChild(this._leftHoverEl);
    this.container.appendChild(this._rightHoverEl);

    this.container.addEventListener("wheel", (ev: WheelEvent) => {
      this.scrollOnWheel(ev);
    });
    this.updateStyle();
  }

  destroy() {
    if (this.container) {
      this.container.classList.remove("scrollable");
      if (this._leftHoverEl)
        this.container.removeChild(this._leftHoverEl);
      if (this._rightHoverEl)
        this.container.removeChild(this._rightHoverEl);

      this.container.removeEventListener("wheel", (ev: WheelEvent) => {
        this.scrollOnWheel(ev);
      });
    }
  }

  updateStyle() {
    this._offset = this.container.offsetWidth + this.container.scrollLeft;
    // console.log(`${this._offset} / ${this.container.scrollWidth}`);
    if (this._offset >= this.container.offsetWidth + 2 && this._leftHoverEl) {
      this._leftHoverEl.classList.remove("hidden");
    }
    else if (this._leftHoverEl) {
      this._leftHoverEl.classList.add("hidden");
    }
    if (this._offset >= this.container.scrollWidth && this._rightHoverEl) {
      this._rightHoverEl.classList.add("hidden");
    }
    else if (this._rightHoverEl) {
      this._rightHoverEl.classList.remove("hidden");
    }
  }

  scroll(vel: number) {
    // console.log("scroll")
    this.container.scrollLeft += vel;
    this.updateStyle();
  }

  scrollOnWheel(e: WheelEvent) {
    if (!this.canScrollOnWheel) return;
    let delta = 0;
    if (e.deltaX > 0 || e.deltaX < 0) {
      delta = e.deltaX;
    } else if (e.deltaY > 0 || e.deltaY < 0) {
      delta = e.deltaY;
    }

    if (delta) {
      this.scroll((delta / 100) * this.velocity * 50);
    }
  }

  scrollOnHover(direction: number) {
    if (!this.canScrollOnHover) return;
    if (this._inertia === direction) {
      // console.log("no change in scroll direction");
      return;
    }
    if (this._inertia != direction && this._handle > -1) {
      // console.log("clear interval", this._handle);
      this.stop();
    }
    console.log(`current direction: ${direction > 0 ? 'right' : 'left'}`);
    this._inertia = direction;
    if (direction > 0) {
      this._handle = setInterval(
        () => this.scroll(this.velocity),
        40
      );
      // console.log("scroll right, set handler to", this._handle);
    } else if (direction < 0) {
      this._handle = setInterval(
        () => this.scroll(-this.velocity),
        40
      );
      // console.log("scroll left, set handler to", this._handle);
    }
  }

  stop() {
    // console.log("stop scrolling.")
    if (this._handle > -1) {
      // console.log("clear interval", this._handle);
      clearInterval(this._handle);
      this._handle = -1;
      this._inertia = 0;
    }
    
  }
}

export default defineComponent({
  inheritAttrs: false,
  props: {
    direction: {
      type: String,
      default: "x",
    },
    scrollOnHover: {
      type: Boolean,
      default: true
    },
    scrollOnWheel: {
      type: Boolean,
      default: true
    },
    speedFactor: {
      type: Number,
      default: 1
    }
  },
  emits: {},
  setup(props, { emit, attrs }) {
    const containerRef = ref(null as HTMLElement | null);
    const hover1Ref = ref(null as HTMLElement | null);
    const hover2Ref = ref(null as HTMLElement | null);

    /**  */
    const hoverAreaWidth = ref(20);
    const handle = ref(-1);
    const inertia = ref(0);

    const offset = ref(0)

    const variables = (object: { style?: Record<string, string> }) =>
      Object.fromEntries(
        Object.entries(object ?? {}).filter(([key, _]) => key.startsWith("--"))
      );

    /** Listeners */
    const onMouseEnter = (e: MouseEvent) => {
        console.log("onMouseEnter")
    }

    const onMouseLeave = (e: MouseEvent) => {
      console.debug("onMouseLeave")
    }

    /** Control */
    const stop = () => {
      console.debug("stop scrolling.")
      if (handle.value > -1) {
        console.debug("clear interval", handle.value);
        clearInterval(handle.value);
        handle.value = -1
        inertia.value = -1
      }
    }

    return {
      containerRef,
      hover1Ref,
      hover2Ref,
      hoverAreaWidth,
      handle,
      inertia,
      offset,
      onMouseEnter,
      onMouseLeave,
      log: (e: any) => console.log(e),
      variables
    };
  },
  computed: {
    useCustomHoverTrigger(): boolean {
      return !!this.$slots.hoverTrigger1 && !!this.$slots.hoverTrigger2
    }
  },
});
</script>

<style lang="scss">
.v3-scrollable {
  &::-webkit-scrollbar {
    width: 0;
    background: transparent;
  }
  overflow: scroll;
  scrollbar-width: none;
  -ms-overflow-style: none;
}
</style>