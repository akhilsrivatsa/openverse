<template>
  <VIconButton
    :tabindex="isTabbable ? 0 : -1"
    class="play-pause"
    :size="buttonSize"
    variant="filled-dark"
    :icon-props="icon === undefined ? undefined : { name: icon, size: iSize }"
    :label="$t(label)"
    :connections="connections"
    @click.stop.prevent="handleClick"
    @mousedown="handleMouseDown"
  >
    <template #default>
      <svg
        v-if="isLoading"
        class="loading p-2"
        :class="`w-${iSize} h-${iSize}`"
        xmlns="http://www.w3.org/2000/svg"
        overflow="visible"
        viewBox="0 0 12 12"
      >
        <circle cx="6" cy="6" r="6" vector-effect="non-scaling-stroke" />
        <path
          d="m 6 0 a 6 6 0 0 1 6 6"
          stroke="white"
          vector-effect="non-scaling-stroke"
        />
      </svg>
    </template>
  </VIconButton>
</template>

<script lang="ts">
import { computed, defineComponent, PropType } from "vue"

import {
  AudioLayout,
  audioLayouts,
  AudioStatus,
  audioStatuses,
  statusVerbMap,
} from "~/constants/audio"
import { defineEvent } from "~/types/emits"
import type { ButtonConnections } from "~/types/button"

import VIconButton from "~/components/VIconButton/VIconButton.vue"

const statusIconMap = {
  playing: "pause",
  paused: "play",
  played: "replay",
  loading: undefined,
} as const

const layoutConnectionsMap: Record<AudioLayout, ButtonConnections> = {
  row: "end",
  global: "all",
  box: "none",
  full: "none",
} as const

/**
 * The mapping of play-pause control sizes to the VIconButton sizes
 * and the sizes of the contained icon.
 */
const sizes = {
  small: { button: "small", icon: 6 },
  medium: { button: "large", icon: 8 },
  large: { button: "larger", icon: 10 },
} as const

/**
 * Displays the control for switching between the playing and paused states of
 * a media file.
 */
export default defineComponent({
  name: "VAudioControl",
  components: { VIconButton },
  model: {
    prop: "status",
    event: "toggle",
  },
  props: {
    /**
     * the current play status of the audio
     */
    status: {
      type: String as PropType<AudioStatus>,
      required: true,
      validator: (val: string) =>
        (audioStatuses as readonly string[]).includes(val),
    },
    /**
     * The size of the button. The size affects both the size of the button
     * itself and the icon inside it.
     */
    size: {
      type: String as PropType<"small" | "medium" | "large">,
      required: true,
      validator: (val: string) => ["small", "medium", "large"].includes(val),
    },
    /**
     * The parent audio layout currently in use. The connections are determined
     * by the layout and the size of the button.
     */
    layout: {
      type: String as PropType<AudioLayout>,
      default: "full",
      validator: (val: string) =>
        (audioLayouts as readonly string[]).includes(val),
    },
    /**
     * Whether the play-pause button can be focused by using the `Tab` key
     */
    isTabbable: {
      type: Boolean,
      default: true,
    },
  },
  emits: {
    toggle: defineEvent<["paused" | "playing"]>(),
  },
  setup(props, { emit }) {
    const isPlaying = computed(() => props.status === "playing")
    const isLoading = computed(() => props.status === "loading")
    /**
     * Get the button label based on the current status of the player.
     */
    const label = computed(() => `playPause.${statusVerbMap[props.status]}`)
    /**
     * Get the button icon based on the current status of the player.
     */
    const icon = computed(() => statusIconMap[props.status])

    /**
     * Set the connections (none-rounded corners) for the button based on the layout.
     */
    const connections = computed(() => {
      return props.layout === "row" && props.size === "small"
        ? "none"
        : layoutConnectionsMap[props.layout]
    })

    /** Convert the `play-pause` sizes to `VIconButton` sizes */
    const buttonSize = computed(() => sizes[props.size].button)

    const iSize = computed(() => sizes[props.size].icon)

    const handleMouseDown = (event: MouseEvent) => {
      if (!props.isTabbable) event.preventDefault() // to prevent focus
    }
    const handleClick = () => {
      emit("toggle", isPlaying.value || isLoading.value ? "paused" : "playing")
    }

    return {
      label,
      icon,
      connections,
      buttonSize,
      iSize,
      isLoading,

      handleClick,
      handleMouseDown,
    }
  },
})
</script>

<style scoped>
@keyframes spinAnimation {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.loading circle,
.loading path {
  fill: transparent;
  stroke-width: 2px;
}

.loading circle {
  stroke: #595258;
}

.loading path {
  stroke-linecap: round;
  transform-origin: 50% 50%;
  animation: spinAnimation 1.4s linear infinite both;
}
</style>
