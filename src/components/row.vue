<template>
  <div class="row" v-if="value" :style="{ 'background-color': `rgba(0,0,0,${5 + 1 * value.level}%)` }">
    <div class="row__insert" :id="value.insertBtnId" @click="onInsert">+</div>
    <div class="row__header" :id="value.id">
      {{ value.type }}
      <div class="row__header__remove" @click="onRemove(value)">-</div>
    </div>
    <div class="row__items" v-if="value.children.length > 0">
      <row v-for="item in value.children" :value="item" :key="item.id" :level="value.level + 1" @remove="onRemove"> </row>
      <div class="row">
        <div class="row__items__add" :id="value.addBtnId" @click="onAdd">增加</div>
      </div>
    </div>
  </div>
</template>

<script>
  import { jsPlumb } from 'jsplumb'
  import { v4 as uuid } from 'uuid'

  export default {
    props: {
      value: {
        type: Object,
        default: null,
      },
    },
    data() {
      return {}
    },
    computed: {},
    watch: {
      value: {
        immediate: true,
        handler() {
          if (this.value) {
            // this.drawLine()
          }
        },
      },
    },
    mounted() {},
    methods: {
      drawLine() {
        this.$nextTick(() => {
          let common = {
            endpoint: 'Blank',
            connector: ['Flowchart'],
          }
          jsPlumb.connect(
            {
              source: this.value.id,
              target: this.value.addBtnId,
              paintStyle: { stroke: '#666666', strokeWidth: 1, dashstyle: '2 2' },
              overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
              anchor: ['Bottom', 'Top'],
            },
            common
          )
          this.value.children.forEach((o) => {
            jsPlumb.connect(
              {
                source: this.value.id,
                target: o.id,
                paintStyle: { stroke: '#0066FF', strokeWidth: 2 },
                overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
                anchor: ['Bottom', 'Top'],
              },
              common
            )
          })
          if (this.level === 1) {
            jsPlumb.connect(
              {
                source: this.value.insertBtnId,
                target: this.value.id,
                paintStyle: { stroke: '#0066FF', strokeWidth: 2 },
                overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
                anchor: ['Bottom', 'Top'],
              },
              common
            )
          }
        })
      },
      onAdd() {},
      onInsert() {},
      onRemove(item) {
        this.$emit('remove', item)
      },
    },
  }
</script>

<style lang="less"></style>
