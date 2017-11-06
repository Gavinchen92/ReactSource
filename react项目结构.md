# React-master项目结构
> React版本 v16.0.0

## 只列出一些关键的文件与文件夹
* eslint-rules  // eslint规则
* fixtures // 供开发者使用的一些测试用实例
* flow // 类型申明
* flow-typed // 类型申明(暂时不明白和前一个文件夹的区别)
* mocks // mock
* packages // 向外输出的包的集合(如: react.js, react-dom.js)
* scripts // 工具
* src // 源码
  * __mocks__ // mock
  * fb // 暂时不明白...
  * isomorphic // 同构层, 可以理解为react.js源码(react顶级api, React.creatElement, React.Children, React.Component)
  * renderers // react-dom, react-native-dom, react-server
  * shared // 共享文件夹, 存放不同路径下公用的方法或变量
  * test // 测试
  * ReactVersion.js // 版本
