<template>
  <div :id="containerId">
    <row v-for="(item, index) in tree" :value="item" :key="index" :index="index" :level="1" @refresh="onRefresh"></row>
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
        tree: [],
        ready: false,
        instance: null,
        containerId: uuid(),
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
      // tree() {
      //   let temp = this.parseExpression(this.tree)
      //   console.log(this.value)
      //   console.log(temp)
      //   console.log('')
      // },
    },
    mounted() {
      this.instance = jsPlumb.getInstance({
        Container: this.containerId,
      })
      this.instance.bind('ready', () => {
        this.ready = true
        this.update()
      })
    },
    methods: {
      update() {
        if (this.value) {
          let tree = this.parseTree(this.value)
          this.injectId(tree)
          this.tree = tree
          this.draw()
        } else {
          this.tree = [
            {
              value: '',
              children: [],
              type: 'unset',
              id: uuid(),
            },
          ]
          this.draw()
        }
      },
      // 注入/更新节点id
      injectId(children = [], level = 1) {
        children.forEach((o, i) => {
          o.id = o.id ? o.id : `id-${uuid()}`
          if (o.type === 'logic') {
            o.addBtnId = `id-${uuid()}`
          }
          o.insertBtnId = `id-${uuid()}`
          o.level = level
          o.index = i
          this.injectId(o.children, level + 1)
        })
      },
      // 树形数据->表达式
      parseExpression(children = [], value = '', level = 1) {
        let exps = []
        children.forEach((o) => {
          if (o.type === 'logic' && o.children.length >= 2) {
            if (level === 1) {
              exps.push(this.parseExpression(o.children, o.value, level + 1))
            } else {
              exps.push(`(${this.parseExpression(o.children, o.value, level + 1)})`)
            }
          } else if (o.type === 'rule' && o.value) {
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
          } else if (expression[i] === '&' || expression[i] === '|') {
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
        let temp = this.parseExpression(this.tree)
        console.log(this.value)
        console.log(temp)
        console.log('')
        if (this.value !== temp) {
          this.$emit('change', temp)
        }

        this.draw()
      },
      draw() {
        this.instance.reset()
        setTimeout(() => {
          this.tree.forEach((o) => {
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
    },
    provide() {
      return {
        share: {
          moveId: '',
        },
      }
    },
  }
</script>

<style lang="less">
  @import '../styles/row.less';
</style>
