<template>
  <div class="row" v-if="value.type !== 'placeholder'" :style="{ 'background-color': `rgba(0,0,0,${5 + 1 * value.level}%)` }">
    <div
      class="row__insert"
      :id="value.insertBtnId"
      @click="onInsert"
      :style="{
        'background-color':
          value.type === 'unset' ||
          !value.value ||
          (value.type === 'logic' && (value.value === '~~' || value.children.filter((o) => o.type !== 'placeholder').length < 2))
            ? '#ddd'
            : undefined,
      }">
      +
    </div>
    <div
      class="row__header"
      :id="value.id"
      :class="{
        'row__header--unset': value.type === 'unset',
        'row__header--empty': value.type === 'rule' && !value.value,
        'row__header--error': value.type === 'logic' && (value.children.filter((o) => o.type !== 'placeholder').length < 2 || value.value === '~~'),
      }"
      @drop="onDrop"
      @dragstart="onDragstart"
      @dragover="onDragover"
      @dragend="onDragend"
      :draggable="
        value.level > 1 && (value.type === 'rule' || (value.type === 'logic' && value.children.filter((o) => o.type === 'placeholder').length === 0))
      ">
      <div class="row__header__ct">
        <span v-if="value.type === 'rule'">
          {{ value.value ? value.value : '设置条件(待实现)' }}
        </span>
        <div v-if="value.type === 'logic' && value.value !== '~~'">
          <span> {{ value.value }} </span><br />
          <span v-if="value.children.filter((o) => o.type !== 'placeholder').length < 2">请设置至少两个条件</span>
        </div>
        <div v-if="value.type === 'unset' || (value.type === 'logic' && value.value === '~~')" style="text-align: left">
          <label :for="value.id"><input :name="value.id" type="radio" @change="onChangeType('logic', '&&')" />&&</label><br />
          <label :for="value.id"><input :name="value.id" type="radio" @change="onChangeType('logic', '||')" />||</label><br />
          <label :for="value.id" v-if="value.type === 'unset'">
            <input :name="value.id" type="radio" @change="onChangeType('rule', Date.now())" />条件</label>
        </div>
      </div>
      <div class="row__header__remove" @click="onRemove" v-if="value.level > 1">-</div>
    </div>
    <div class="row__items">
      <row v-for="item in value.children" :value="item" :key="item.id" :level="value.level + 1" :parent="value" @refresh="$emit('refresh')"> </row>
      <div class="row" v-if="value.type === 'logic'">
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
      parent: {
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
      onAdd() {
        let index = this.value.children.findIndex((o) => o.type === 'placeholder')
        if (index >= 0) {
          this.value.children.splice(index, 1)
        }
        this.value.children.push({
          value: '?',
          children: [],
          type: 'unset',
          id: uuid(),
        })
        this.$emit('refresh')
      },
      onInsert() {
        if (this.value.type !== 'unset' && this.value.type !== 'unset-logic') {
          let index = this.parent.children.findIndex((o) => o.id === this.value.id)
          this.parent.children.splice(index, 1)
          let node = {
            value: '~~',
            children: [this.value],
            type: 'logic',
            id: uuid(),
          }
          debugger
          this.parent.children.push(node)
          // this.value.children = [Object.assign({}, this.value)]
          // this.value.value = '#'
          // this.value.type = 'unset'
          // this.value.id = uuid()

          this.$emit('refresh')
        }
      },
      onRemove() {
        let index = this.parent.children.findIndex((o) => o.id === this.value.id)
        if (index >= 0) {
          this.parent.children.splice(index, 1)
          if (this.parent.children.length < 2) {
            this.parent.children.splice(0, 0, { value: '#', children: [], type: 'placeholder' })
          }
          this.$emit('refresh')
        }
      },
      onChangeType(type, value) {
        this.value.type = type
        this.value.value = value
        if (this.value.type === 'logic') {
          this.value.addBtnId = `id-${uuid()}`
        }
        this.$emit('refresh')
      },
      onDragstart(e) {
        e.dataTransfer.setData('node', JSON.stringify(this.value))
      },
      onDrop(e) {
        if (this.value.type === 'logic') {
          let node = JSON.parse(e.dataTransfer.getData('node'))
          this.value.children.push(node)
          this.share.moveId = node.id
        }
      },
      onDragover(e) {
        e.preventDefault()
      },
      onDragend(e) {
        if (this.share.moveId) {
          this.onRemove()
          this.share.moveId = ''
        }
      },
    },
    inject: ['share'],
  }
</script>

<style lang="less"></style>
