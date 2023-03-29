# 文档工具

### 用途：
1. 做笔记
2. 写个人博客
3. 写开源项目文档

### 功能：
1. 静态托管github，不需要购买服务器，不需要搭建数据库，不需要懂前后端
2. 支持Markdown、样式丰富

### 方案：
> 推荐 docsify

|        | [VuePress](https://vuepress.vuejs.org/zh/guide/)        | [docsify](https://docsify.js.org/#/zh-cn/quickstart)         | [Hexo](https://hexo.io/zh-cn/docs/)                         | [GitBook](https://chrisniael.gitbooks.io/gitbook-documentation/content/) | [teedoc](https://teedoc.github.io/get_started/zh/)           |
| ------ | ------------------------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 活跃度 | [github star: 21.3k](https://github.com/vuejs/vuepress) | [github star: 22.9k](https://github.com/docsifyjs/docsify)   | [github star: 36.3k](https://github.com/hexojs/hexo/issues) | [github star: 25.4k](https://github.com/GitbookIO/gitbook)   | [github star：83](https://github.com/teedoc/teedoc)          |
| 效果   | http://10.11.40.34:8080/                                | [http://10.11.40.34:3000/](http://10.11.40.34:3000/#/?id=数据清洗) |                                                             |                                                              | http://10.11.40.34:2333/doc1/                                |
| 扩展性 | 可自定义主题、路由、外链                                |                                                              |                                                             |                                                              |                                                              |
| 优点   | 支持目录、搜索、国际化、分模块                          | 使用Markdown配置，功能齐全，定制化程度高                     | 老牌的静态文档生成工具，功能强大，主题丰富                  | 轻量、简单，顶部分模块、左侧多层级、搜索                     | 轻量、简单，顶部分模块、左侧多层级、搜索、国际化、夜间模式、外链 |
| 缺点   | 配置复杂，大量JavaScript配置，对后端不友好              |                                                              | 相对有点复杂，有点重，适合建站、博客、文档等多种场景        | 文档太多时重新加载很慢，导航结构有限制                       | 社区活跃低                                                   |