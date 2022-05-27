<template>
  <div class="text-6xl sm:text-7xl md:text-8xl lg:text-9xl w-fit">
    <div
      v-for="(row, rindex) in board"
      :key="rindex"
      class="flex flex-row justify-center"
    >
      <div
        v-for="(square, cindex) in row"
        :key="cindex"
        :class="{ 'border-blue-400': enabled, 'border-b-2': rindex < 2, 'border-r-2': cindex < 2 }"
        class="
          border-gray-400
          w-20
          h-20
          sm:w-24 sm:h-24
          md:w-32 md:h-32
          lg:w-40 lg:h-40
          flex
          justify-center
          items-center
        "
      >
        <button
          @click="markSquare(rindex, cindex)"
          class="h-full w-full"
          :disabled="square !== '' || !enabled"
        >
          <div
            :class="{ hidden: square !== '' || !enabled }"
            class="
              h-full
              w-full
              flex
              justify-center
              items-center
              opacity-0
              hover:opacity-20
              transition-opacity
            "
          >
            {{ marker }}
          </div>
          {{ square }}
        </button>
      </div>
    </div>
    <div class="border-blue-400 border-2 hidden"></div>
  </div>
</template>

<script lang="ts">
import type { PropType } from "vue";

export default {
  name: "GameBoard",
  props: {
    board: {
      type: Array as PropType<Array<string>>,
      default: () => [
        ["", "", ""],
        ["", "", ""],
        ["", "", ""],
      ],
    },
    enabled: {
      type: Boolean as PropType<boolean>,
      default: false,
    },
    marker: {
      type: String as PropType<string>,
      default: "",
    },
  },
  data() {
    return {
      squareClasses:
        "border-grey-800 border-2 w-40 h-40 flex justify-center items-center",
    };
  },
  methods: {
    markSquare(row: number, col: number) {
      this.$emit("mark-square", [row, col]);
    },
  },
};
</script>

