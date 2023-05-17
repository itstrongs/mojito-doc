# 前端

## 技术栈
* 基础：html、[javascript](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)、css

* javascript 环境：node

* 包管理器：npm、yarn、pnpm

* 框架：[vue](https://v3.cn.vuejs.org/guide/introduction.html)、[react](https://zh-hans.reactjs.org/)、angular

* UI 框架：[ant design](https://ant.design/index-cn)、[element ui](https://element.eleme.cn/#/zh-CN/component/installation)、[element plus](https://element-plus.org/zh-CN/guide/design.html)、fusion

* 打包构建：[vite](https://cn.vitejs.dev/)、[electron](https://www.electronjs.org/zh/docs/latest)

* 工具库：axios

* 扩展语言：typescript、scss、less、stylus

* vue 生态：vuex、vue-router

目前技术选型最佳实践：React + Vite + Ant Design

## 学习资料
* [MDN](https://developer.mozilla.org/zh-CN/)

* 掘金

## 常见问题
### npm install 报错
可能是 npm 太新了，和当前项目不匹配，尝试换到低版本，比如`v14.15.4`

### 其他
```bash
# Error: EACCES: permission denied
npm install --unsafe-perm=true --allow-root

# note 报错时清理 cache 重试
rm -rf node_modules
npm cache clean --force
rm package-lock.json

# 找不到模块“path”或其相应的类型声明。
npm install @types/node --save-dev
```