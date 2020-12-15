<template>
  <div class="row" v-if="value">
    <div class="row__header" :id="value.id">
      {{ value.type }}
    </div>
    <div class="row__items">
      <row
        v-for="(item, index) in value.children"
        :value="item"
        :key="index"
        :level="level + 1"
      ></row>
    </div>
  </div>
</template>

<script>
import { jsPlumb } from "jsplumb";

export default {
  props: {
    value: {
      type: Object,
      default: null,
    },
    level: {
      type: Number,
      default: 0,
    },
  },
  data() {
    return {};
  },
  watch: {
    value: {
      immediate: true,
      handler() {
        if (this.value) {
          this.$nextTick(() => {
            this.value.children.forEach((o) => {
              jsPlumb.connect({
                source: this.value.id,
                target: o.id,
                endpoint: "Blank",
                connector: ["Straight"],
                overlays: [["Arrow", { width: 8, length: 8, location: 1 }]],
                anchor: ["Bottom", "Top"],
              });
            });
          });
        }
      },
    },
  },
  mounted() {},
  methods: {},
};
</script>

<style lang="less">
</style>