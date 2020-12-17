<template>
  <div id="app">
    <div>构建树形结构数据，渲染图形：</div>
    <div>{ type: '条件2/并/或', children: []}</div>
    <hr />
    <div>表达式：</div>
    <div>
      <input type="text" v-model="expression" style="width: 100%" />
    </div>
    <hr />
    <rows v-model="expression"></rows>
  </div>
</template>

<script>
  import Vue from 'vue'
  import { jsPlumb } from 'jsplumb'
  import { v4 as uuid } from 'uuid'

  import rows from './components/rows'

  Vue.component('rows', rows)

  export default {
    name: 'App',
    data() {
      return {
        expression: localStorage.getItem('expression'),
      }
    },
    computed: {},
    watch:{
      expression(){
        localStorage.setItem('expression',this.expression)
      }
    },
    mounted() {
      let test = [
        {
          type: '并',
          children: [
            {
              type: '条件1',
              children: [],
            },
            {
              type: '或',
              children: [
                {
                  type: '并',
                  children: [
                    {
                      type: '条件2',
                      children: [],
                    },
                    {
                      type: '条件3',
                      children: [],
                    },
                    {
                      type: '条件4',
                      children: [],
                    },
                  ],
                },
                {
                  type: '条件5',
                  children: [],
                },
                {
                  type: '并',
                  children: [
                    {
                      type: '条件6',
                      children: [],
                    },
                    {
                      type: '条件7',
                      children: [],
                    },
                  ],
                },
              ],
            },
            {
              type: '并',
              children: [
                {
                  type: '并',
                  children: [
                    {
                      type: '条件8',
                      children: [],
                    },
                    {
                      type: '条件9',
                      children: [],
                    },
                  ],
                },
                {
                  type: '条件10',
                  children: [],
                },
              ],
            },
          ],
        },
      ]
      this.expression = this.expression?this.expression:this.parseExpression(test).replace(/^\(/, '').replace(/^\(/, '').replace(/\)$/, '').replace(/\)$/, '')
    },
    methods: {
      // 树形数据->表达式
      parseExpression(children = [], type = '并') {
        let exps = []
        children.forEach((o) => {
          if (/[并或]/.test(o.type)) {
            exps.push(this.parseExpression(o.children, o.type))
          } else {
            exps.push(o.type)
          }
        })
        return `(${exps.join({ 并: '&&', 或: '||' }[type])})`
      },
    },
  }
</script>

<style lang="less">
  @import './styles/row.less';
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;

    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
  }
</style>
