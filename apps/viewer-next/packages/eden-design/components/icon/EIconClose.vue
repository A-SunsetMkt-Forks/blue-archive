<script setup lang="ts">
import EIconAdd from "./EIconAdd.vue";
import type { IconProps } from "../types/EdenIcon/IconProps";
import { parseColor } from "../../_utils/colorUtils";

withDefaults(defineProps<IconProps>(), {
  size: "16",
  color: "#616161",
  strokeWidth: "0",
  fill: false,
  rotate: 45,
});
</script>

<template>
  <e-icon-add
    :rotate="rotate"
    :size="size"
    :color="parseColor(color)"
    :stroke-width="strokeWidth"
    :fill="fill"
  />
</template>
