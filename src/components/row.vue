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
        'row__header--disabled': status === 'disabled',
      }"
      @drop="onDrop"
      @dragstart="onDragstart"
      @dragover="onDragover"
      @dragend="onDragend"
      :draggable="value.level > 1">
      <!-- (value.type === 'rule' || (value.type === 'logic' && value.children.filter((o) => o.type === 'placeholder').length === 0)) -->
      <div class="row__header__ct">
        <span v-if="value.type === 'rule'">
          {{ value.value ? value.value : '设置条件(待实现)' }}
        </span>
        <div v-if="value.type === 'logic' && value.value !== '~~'">
          <span> {{ value.value }} </span>
          <button @click="onSwitch">切换</button><br />
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
      return {
        status: '',
      }
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
          parents: [...this.value.parents],
          type: 'unset',
          id: uuid(),
        })
        this.$emit('refresh')
      },
      onInsert() {
        if (this.value.type !== 'unset' && this.value.type !== 'unset-logic') {
          if (this.parent) {
            let index = this.parent.children.findIndex((o) => o.id === this.value.id)
            let id = uuid()
            this.value.parents.push(id)
            let node = {
              value: '~~',
              children: [this.value],
              parents: [...this.value.parents],
              type: 'logic',
              id,
            }
            this.parent.children.splice(index, 1, node)
            this.$emit('refresh')
          }
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
        this.share.moveId = this.value.id
      },
      onDrop(e) {
        // TODO: 拖动左右移动
        if (this.value.type === 'logic') {
          let node = JSON.parse(e.dataTransfer.getData('node'))
          if (this.value.id !== node.id && !this.value.parents.includes(node.id)) {
            this.value.children.push(node)
            this.share.dropId = this.value.id
          }
        }
        this.status = ''
      },
      onDragover(e) {
        e.preventDefault()
        if (this.value.type === 'logic') {
          if (this.value.parents.includes(this.share.moveId)) {
            this.status = 'disabled'
            return
          }
        }
        this.status = ''
      },
      onDragend(e) {
        if (this.share.dropId) {
          this.onRemove()
          this.share.dropId = ''
        }
        this.status = ''
      },
      onSwitch() {
        this.value.value = this.value.value === '&&' ? '||' : '&&'
      },
    },
    inject: ['share'],
  }
</script>

<style lang="less"></style>
