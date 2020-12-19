<template>
  <div
    class="rows"
    ref="rows"
    @mousewheel="onMousewheel"
    @mousemove="onMouseover"
    @mousedown="onMousedown"
    @mouseup="onMouseup"
    @mouseleave="onMouseleave">
    ctWidth:{{ ctWidth }}
    <br />
    rowsWidth:{{ rowsWidth }}
    <br />
    overX:{{ overX }}
    <div
      class="rows__ct"
      ref="ct"
      :id="containerId"
      :style="{ transform: `scale(${1 + zoom})`, top: `${top}px`, left: `${left}px`, opacity: inited ? 1 : 0 }">
      <row v-for="(item, index) in tree.children" :value="item" :key="index" :index="index" :level="1" :parent="tree" @refresh="onRefresh"></row>
    </div>
  </div>
</template>

<script>
  import Vue from 'vue'
  import { jsPlumb } from 'jsplumb'
  import { v4 as uuid } from 'uuid'
  import row from './row'
  Vue.component('row', row)

  export default {
    model: {
      prop: 'value',
      event: 'change',
    },
    props: {
      value: {
        type: String,
        default: '',
      },
    },
    data() {
      return {
        tree: {
          parents: [],
          id: uuid(),
          type: 'root',
          children: [
            {
              value: '?',
              children: [],
              parents: [],
              type: 'unset',
              id: uuid(),
            },
          ],
        },
        ready: false,
        instance: null,
        containerId: uuid(),
        //
        zoom: 0,
        top: 0,
        left: 0,
        overX: 0,
        overY: 0,
        startX: 0,
        startY: 0,
        moveX: 0,
        moveY: 0,
        lastLeft: 0,
        lastTop: 0,
        isDown: false,
        ctTop: 0,
        ctLeft: 0,
        ctWidth: 0,
        ctHeight: 0,
        rowsTop: 0,
        rowsLeft: 0,
        rowsWidth: 0,
        rowsHeight: 0,
        //
        inited: false,
      }
    },
    computed: {},
    watch: {
      value: {
        immediate: true,
        handler(v, o) {
          if (this.ready && v !== o) {
            this.update()
          }
        },
      },
      inited() {
        if (this.inited) {
          setTimeout(() => {
            this.updateCtSize()
            this.fitSize()
          })
        }
      },
    },
    mounted() {
      this.instance = jsPlumb.getInstance({
        Container: this.containerId,
      })
      this.instance.bind('ready', () => {
        this.ready = true
        this.update()
      })

      // 监听图形尺寸变化

      new ResizeObserver(() => {
        this.updateCtSize()
      }).observe(this.$refs.ct)
      //   setInterval(() => {
      //     this.updateCtSize()
      //   }, 1000)
      // 监听容器尺寸变化
      this.updateRowsSize()
      new ResizeObserver(() => {
        this.updateRowsSize()
      }).observe(this.$refs.rows)
    },
    methods: {
      update() {
        if (this.value) {
          let tree = {
            children: this.parseTree(this.value),
            parents: [],
            id: uuid(),
            type: 'root',
          }
          this.injectId(tree)
          this.tree = tree

          this.draw()
        } else {
          this.tree = {
            parents: [],
            id: uuid(),
            type: 'root',
            children: [
              {
                value: '?',
                children: [],
                parents: [],
                type: 'unset',
                id: uuid(),
              },
            ],
          }
          this.draw()
        }
        this.inited = true
      },
      // 注入/更新节点id
      injectId(node, level = 1) {
        node.children.forEach((o, i) => {
          if (o.type === 'logic') {
            o.addBtnId = uuid()
          }
          if (o.type !== 'placeholder') {
            o.id = o.id ? o.id : uuid()
            o.insertBtnId = uuid()
          }

          o.parents = node.parents.concat([node.id])

          o.level = level
          o.index = i
          this.injectId(o, level + 1)
        })
      },
      // 树形数据->表达式
      parseExpression(children = [], value = '', level = 1) {
        let exps = []
        children.forEach((o) => {
          if (o.type === 'logic') {
            if (o.children.length >= 2) {
              if (level === 1) {
                exps.push(this.parseExpression(o.children, o.value, level + 1))
              } else {
                exps.push(`(${this.parseExpression(o.children, o.value, level + 1)})`)
              }
            } else {
              let placeholder = []
              if (o.children.length === 0) {
                placeholder = [
                  { value: '#', children: [], type: 'placeholder', parents: o.parents.concat([o.id]) },
                  { value: '#', children: [], type: 'placeholder', parents: o.parents.concat([o.id]) },
                ]
              } else if (o.children.length === 1) {
                placeholder = [{ value: '#', children: [], type: 'placeholder', parents: o.parents.concat([o.id]) }]
              }
              if (level === 1) {
                exps.push(this.parseExpression([...placeholder, ...o.children], o.value, level + 1))
              } else {
                exps.push(`(${this.parseExpression([...placeholder, ...o.children], o.value, level + 1)})`)
              }
            }
          } else if (o.type === 'rule' && o.value) {
            exps.push(o.value)
          } else if (o.type === 'placeholder') {
            exps.push(o.value)
          } else if (o.type === 'unset') {
            exps.push(o.value)
          } else if (o.type === 'unset-logic') {
            exps.push(o.value)
          }
        })
        return `${exps.join(value)}`
      },
      // 表达式->树形数据
      parseTree(expression) {
        let records = this.parseTreeRecord(expression)
        let node = this.parseTreeNode(records)
        return node ? [node] : []
      },
      parseTreeNode(records) {
        let logic = records.find((o) => o.type === 'logic')
        if (logic) {
          let other = records.filter((o) => o.type !== 'logic')
          let node = {
            value: logic.expr,
            children: [],
            type: 'logic',
          }
          other.forEach((o) => {
            if (o.type === 'next') {
              node.children.push(this.parseTreeNode(o.children))
            } else if (o.type === 'unset') {
              node.children.push({
                value: '?',
                children: [],
                type: 'unset',
              })
            } else if (o.type === 'placeholder') {
              node.children.push({
                value: '#',
                children: [],
                type: 'placeholder',
              })
            } else {
              node.children.push({
                value: o.expr,
                children: [],
                type: 'rule',
              })
            }
          })
          return node
        } else {
          return records.length > 0 ? this.parseTreeNode(records[0].children) : null
        }
      },
      parseTreeRecord(expression) {
        let ruleStack = []
        let ruleArray = []
        for (let i = 0; i < expression.length; i++) {
          if (expression[i] === '(') {
            let leftStack = []
            let rightStack = []
            let k = i
            for (; k < expression.length; k++) {
              if (expression[k] === '(') {
                // 左符号位置记录
                leftStack.push(k)
              } else if (expression[k] === ')') {
                // 右符号位置记录
                rightStack.push(k)
                // 寻找到括号快
                if (leftStack.length === rightStack.length) {
                  // 出栈
                  let left = leftStack.shift()
                  let right = rightStack.pop()
                  leftStack = []
                  rightStack = []
                  let next = expression.substring(left + 1, right)
                  ruleArray.push({
                    expr: next,
                    type: 'next',
                    children: this.parseTreeRecord(next),
                  })
                  break
                }
              }
            }
            i = k
          } else if (expression[i] === '&' || expression[i] === '|' || expression[i] === '~') {
            if (ruleStack.length > 0) {
              ruleArray.push({
                expr: ruleStack.join(''),
                type: 'rule',
                children: [],
              })
            }

            ruleArray.push({
              expr: expression[i] + expression[i],
              type: 'logic',
              children: [],
            })
            ruleStack = []
            i += 1
          } else if (expression[i] === '#') {
            ruleArray.push({
              expr: expression[i],
              type: 'placeholder',
              children: [],
            })
          } else if (expression[i] === '?') {
            ruleArray.push({
              expr: expression[i],
              type: 'unset',
              children: [],
            })
          } else {
            ruleStack.push(expression[i])
            if (i === expression.length - 1) {
              ruleArray.push({
                expr: ruleStack.join(''),
                type: 'rule',
                children: [],
              })
            }
          }
        }
        return ruleArray
      },
      // 重画线
      onRefresh() {
        let temp = this.parseExpression(this.tree.children)
        if (this.value !== temp) {
          this.$emit('change', temp)
        }

        this.draw()
      },
      draw() {
        this.instance.reset()
        setTimeout(() => {
          this.tree.children.forEach((o) => {
            this.drawLine(o)
          })
        })
      },
      drawLine(item, level = 1) {
        let common = {
          endpoint: 'Blank',
          connector: ['Flowchart'],
        }
        if (item.addBtnId) {
          this.instance.connect(
            {
              source: item.id,
              target: item.addBtnId,
              paintStyle: { stroke: '#666666', strokeWidth: 1, dashstyle: '2 2' },
              overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
              anchor: ['Bottom', 'Top'],
            },
            common
          )
        }

        item.children.forEach((o) => {
          if (o.type !== 'placeholder') {
            this.instance.connect(
              {
                source: item.id,
                target: o.id,
                paintStyle: { stroke: '#0066FF', strokeWidth: 2 },
                overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
                anchor: ['Bottom', 'Top'],
              },
              common
            )
            this.drawLine(o, level + 1)
          }
        })
        if (level === 1) {
          this.instance.connect(
            {
              source: item.insertBtnId,
              target: item.id,
              paintStyle: { stroke: '#0066FF', strokeWidth: 2 },
              overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
              anchor: ['Bottom', 'Top'],
            },
            common
          )
        }
      },
      //
      onMousewheel(e) {
        if (e.ctrlKey) {
          if (e.deltaY < 0) {
            if (this.zoom < 2.9) {
              this.changeZoom(0.1)
            }
          } else if (e.deltaY > 0) {
            if (this.zoom > -0.8) {
              this.changeZoom(-0.1)
            }
          }
        } else {
          this.left -= e.deltaX
          this.top -= e.deltaY
          this.lastLeft = this.left
          this.lastTop = this.top
        }
        e.preventDefault()
      },
      changeZoom(value) {
        let lastCtWidth = this.ctWidth * (1 + this.zoom)
        let lastCtHeight = this.ctHeight * (1 + this.zoom)
        let lastOffsetX = this.overX - this.lastLeft - this.rowsLeft
        let lastOffsetY = this.overY - this.lastTop - this.rowsTop
        let rateX = lastOffsetX / lastCtWidth
        let rateY = lastOffsetY / lastCtHeight

        this.zoom += value

        let newCtWidth = this.ctWidth * (1 + this.zoom)
        let newCtHeight = this.ctHeight * (1 + this.zoom)
        let moreWidth = newCtWidth - lastCtWidth
        let moreHeight = newCtHeight - lastCtHeight
        let newSpanX = newCtWidth * rateX - lastOffsetX
        let newSpanY = newCtHeight * rateY - lastOffsetY

        this.left = this.lastLeft - newSpanX
        this.top = this.lastTop - newSpanY
        this.lastLeft = this.left
        this.lastTop = this.top
      },
      onMouseover(e) {
        this.overX = e.clientX
        this.overY = e.clientY

        if (this.isDown) {
          this.moveX = e.clientX
          this.moveY = e.clientY
          this.left = this.lastLeft + this.moveX - this.startX
          this.top = this.lastTop + this.moveY - this.startY
        }
      },
      onMousedown(e) {
        this.startX = e.clientX
        this.startY = e.clientY
        this.isDown = true
      },
      onMouseup(e) {
        this.isDown = false
        this.lastLeft = this.left
        this.lastTop = this.top
      },
      onMouseleave() {
        this.isDown = false
        this.lastLeft = this.left
        this.lastTop = this.top
      },
      updateCtSize() {
        if (this.$refs.ct) {
          let { top, left, width, height } = this.$refs.ct.getBoundingClientRect()

          this.ctTop = top
          this.ctLeft = left
          this.ctWidth = this.$refs.ct.offsetWidth // width / (1 + this.zoom)
          this.ctHeight = this.$refs.ct.offsetHeight // height / (1 + this.zoom)
        }
      },
      updateRowsSize() {
        if (this.$refs.rows) {
          let { top, left, width, height } = this.$refs.rows.getBoundingClientRect()
          this.rowsTop = top
          this.rowsLeft = left
          this.rowsWidth = this.$refs.rows.offsetWidth //width
          this.rowsHeight = this.$refs.rows.offsetHeight // height
        }
      },
      fitSize() {
        this.rateRows = this.rowsWidth / this.rowsHeight
        this.rateCt = this.ctWidth / this.ctHeight
        let zoom = 0
        let left = 0
        let top = 0
        if (this.rateRows > this.rateCt) {
          zoom = this.rowsHeight / this.ctHeight - 1
        } else if (this.rateRows < this.rateCt) {
          zoom = this.rowsWidth / this.ctWidth - 1
        }
        zoom = (zoom > 0 ? Math.floor(zoom * 10) : Math.ceil(zoom * 10)) / 10
        left = (this.rowsWidth - this.ctWidth * (1 + zoom)) / 2

        top = (this.rowsHeight - this.ctHeight * (1 + zoom)) / 2

        this.overX = this.rowsLeft
        this.overY = this.rowsTop

        this.changeZoom(zoom)

        this.onMousedown({
          clientX: 0,
          clientY: 0,
        })
        this.onMouseover({
          clientX: left,
          clientY: top,
        })
        this.onMouseup()
      },
    },
    provide() {
      return {
        share: {
          moveId: '',
          dropId: '',
        },
      }
    },
  }
</script>

<style lang="less">
  @import '../styles/row.less';
</style>
