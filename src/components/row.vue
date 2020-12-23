<template>
  <!-- 'background-color': `rgba(0,0,0,${5 + 1 * value.level}%)` -->
  <div class="row" v-if="value.type !== 'placeholder'">
    <div
      class="row__insert"
      :id="value.insertBtnId"
      @click="onInsert"
      @mousedown.stop
      @mouseup.stop
      :style="{
        visibility: value.type === 'unset' ? 'hidden' : undefined,
      }">
      <!-- (value.type === 'logic' && (value.value === '~~' || value.children.filter((o) => o.type !== 'placeholder').length < 2)) -->
      <i class="el-icon-circle-plus"></i>
    </div>
    <div
      class="row__header"
      :id="value.id"
      :class="{
        'row__header--unset': value.type === 'unset',
        'row__header--empty': value.type === 'rule' && !value.value,
        'row__header--error':
          value.type === 'unset' ||
          (value.type === 'logic' && (value.children.filter((o) => o.type !== 'placeholder').length < 2 || value.value === '~~')),
        'row__header--drop': status === 'drop',
        'row__header--disabled': status === 'disabled',
        //
        'row__header--rule': value.type === 'rule',
        'row__header--b': value.value === '&&',
        'row__header--h': value.value === '||',
        //
        'row__header--drag': value.level > 1 && value.id === current,
      }"
      @drop="onDrop"
      @dragstart="onDragstart"
      @dragover="onDragover"
      @dragend="onDragend"
      @dragleave="onDragleave"
      :draggable="value.level > 1 && value.id === current"
      @mousedown.stop
      @mouseup.stop
      @click.stop="onClick"
      :style="{
        'margin-bottom': `${getMarginBottom}px`,
      }">
      <div class="row__header__ct" :class="{ 'row__header__ct--focus': value.id === current }" @dragover.stop="onDragover">
        <span v-if="value.type === 'rule'">
          {{ value.value }}
        </span>
        <div
          v-if="value.type === 'unset-rule'"
          @click="
            value.value = Date.now()
            value.type = 'rule'
            $emit('refresh')
          ">
          <section class="row__header__ct__error"><i class="el-icon-warning"></i><span>请设置条件</span></section>
        </div>
        <div v-if="value.type === 'logic' && value.value !== '~~'">
          <span> {{ parseLabel(value.value) }} </span>
          <el-button type="text" @click="onSwitch">
            <i class="el-icon-sort" style="transform: rotate(90deg)"></i>
          </el-button>
          <section class="row__header__ct__error" v-if="value.children.filter((o) => o.type !== 'placeholder').length < 2">
            <i class="el-icon-warning"></i><span>请设置至少两个条件</span>
          </section>
        </div>
        <div v-if="value.type === 'unset' || (value.type === 'logic' && value.value === '~~')" style="text-align: left">
          <el-radio v-model="unsetSelect" :label="{ type: 'logic', value: '&&' }">满足所有条件</el-radio>
          <br />
          <el-radio v-model="unsetSelect" :label="{ type: 'logic', value: '||' }">满足任一条件</el-radio>
          <br />
          <el-radio v-model="unsetSelect" :label="{ type: 'rule', value: '*' }" v-if="value.type === 'unset'">条件</el-radio>
          <section class="row__header__ct__error" v-if="value.children.filter((o) => o.type !== 'placeholder').length < 2">
            <i class="el-icon-warning"></i><span>请选择节点类型</span>
          </section>
        </div>
        <i class="row__header__ct__color-bar" @dragover.stop="onDragover"> </i>
        <el-button type="text" class="row__header__ct__remove" @click="onRemove" v-if="value.level > 1" @dragover.stop="onDragover">
          <i class="el-icon-remove"></i>
        </el-button>
      </div>
    </div>
    <div class="row__items">
      <row
        v-for="item in value.children"
        :value="item"
        :key="item.id"
        :level="value.level + 1"
        :parent="value"
        @refresh="
          (noDraw) => {
            $emit('refresh', noDraw)
          }
        "
        @focus="onFocus"
        :current="current">
      </row>
      <div class="row" v-if="value.type === 'logic'">
        <div class="row__items__add" :id="value.addBtnId" @click="onAdd" @mousedown.stop @mouseup.stop>
          <i class="el-icon-plus"></i>
        </div>
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
      current: {
        type: String,
        default: '',
      },
    },
    data() {
      return {
        status: '',
        unsetSelect: null,
      }
    },
    computed: {
      getMarginBottom() {
        // value.type !== 'unset' && parent.children.findIndex((o) => o.type === 'unset') >= 0
        //   ? 60
        //   : value.type === 'logic' &&
        //     value.children.filter((o) => value.type !== 'placeholder').length >= 2 &&
        //     value.value !== '~~' &&
        //     parent.children.findIndex(
        //       (o) => o.type === 'logic' && (o.children.filter((o) => o.type !== 'placeholder').length < 2 || o.value === '~~')
        //     ) >= 0
        //   ? 200
        //   : 0
        // if (this.value.type !== 'unset' && this.parent.children.findIndex((o) => o.type === 'unset') >= 0) {
        //   if (this.value.type === 'logic' && this.value.children.filter((o) => o.type !== 'placeholder').length < 2) {
        //     return 112.4 - 69.2
        //   }
        //   return 112.4 - 47.6
        // }
        //else if (this.value.type === 'logic') {
        //   if (this.value.value !== '~~' && this.value.children.filter((o) => value.type !== 'placeholder') >= 2) {
        //     return 0
        //   }
        // }
        return 0
      },
    },
    watch: {
      value: {
        immediate: true,
        handler() {
          if (this.value) {
            // this.drawLine()
          }
        },
      },
      unsetSelect() {
        if (this.unsetSelect) {
          this.value.type = this.unsetSelect.type
          this.value.value = this.unsetSelect.value
          if (this.value.type === 'logic') {
            this.value.addBtnId = `id-${uuid()}`
          }
          this.$emit('refresh')
        }
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
        if (this.value.type !== 'unset') {
          if (this.parent) {
            let index = this.parent.children.findIndex((o) => o.id === this.value.id)
            let id = uuid()
            this.value.parents.push(id)
            let node = {
              value: '~~',
              children: [this.value],
              parents: [...this.value.parents],
              type: 'logic',
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
      onDragstart(e) {
        e.dataTransfer.setData('node', JSON.stringify(this.value))
        this.share.moveId = this.value.id
      },
      onDrop(e) {
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
        clearTimeout(this.timer)
        if (this.value.type === 'logic') {
          if (this.value.parents.includes(this.share.moveId) || this.value.children.findIndex((o) => o.id === this.share.moveId) >= 0) {
            this.status = 'disabled'
            return
          }
        } else if (this.value.type === 'rule' && this.value.id !== this.share.moveId) {
          this.status = 'disabled'
          return
        } else if (this.value.type === 'unset' && this.value.id !== this.share.moveId) {
          this.status = 'disabled'
          return
        }
        if (this.value.id !== this.share.moveId) {
          this.status = 'drop'
        }
      },
      onDragleave(e) {
        clearTimeout(this.timer)
        this.timer = setTimeout(() => {
          this.status = ''
        }, 200)
      },
      onDragend(e) {
        if (this.share.dropId) {
          this.onRemove()
          this.share.dropId = ''
        }
      },
      onSwitch() {
        this.value.value = this.value.value === '&&' ? '||' : '&&'
        this.$emit('refresh', true)
      },
      parseLabel(value) {
        if (value === '&&') {
          return '满足所有条件'
        } else if (value === '||') {
          return '满足任一条件'
        } else {
          return value
        }
      },
      onClick() {
        this.$emit('focus', this.value)
      },
      onFocus(row) {
        this.$emit('focus', row)
      },
    },
    inject: ['share'],
  }
</script>

<style lang="less"></style>
