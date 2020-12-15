<template>
  <div id="app">
    <div>等效表达式：{{ expression }}</div>
    <div class="row">
      <row
        v-for="(item, index) in tree"
        :value="item"
        :key="index"
        :level="1"
      ></row>
    </div>
  </div>
</template>

<script>
import Vue from "vue";
import { jsPlumb } from "jsplumb";
import { v4 as uuid } from "uuid";

import row from "./components/row";

Vue.component("row", row);

export default {
  name: "App",
  data() {
    return {
      tree: [],
      expression: "",
    };
  },
  computed: {},
  mounted() {
    jsPlumb.bind("ready", () => {
      this.tree = [
        {
          id: uuid(),
          type: "并",
          children: [
            {
              id: uuid(),
              type: "条件",
              children: [],
            },
            {
              id: uuid(),
              type: "或",
              children: [
                {
                  id: uuid(),
                  type: "条件",
                  children: [],
                },
                {
                  id: uuid(),
                  type: "条件",
                  children: [],
                },
                {
                  id: uuid(),
                  type: "并",
                  children: [
                    {
                      id: uuid(),
                      type: "条件",
                      children: [],
                    },
                    {
                      id: uuid(),
                      type: "条件",
                      children: [],
                    },
                  ],
                },
              ],
            },
            {
              id: uuid(),
              type: "并",
              children: [
                {
                  id: uuid(),
                  type: "条件",
                  children: [],
                },
                {
                  id: uuid(),
                  type: "条件",
                  children: [],
                },
              ],
            },
          ],
        },
      ];
    });
  },
  methods: {
    parseExpression() {
      let exps = [];
      this.tree.forEach((o) => {
        if (o.type === "条件") {
          exps.push(o.type);
        } else {
          exps.push(this.parseExpression(o.children));
        }
      });
      return exps;
    },
  },
};
</script>

<style lang="less">
@import "./styles/row.less";
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
