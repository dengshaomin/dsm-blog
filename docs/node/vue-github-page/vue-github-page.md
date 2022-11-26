github上创建一个repo

使用[vue cli](https://cli.vuejs.org/zh/guide/creating-a-project.html)创建一个项目,新建一个分支gh-pages并推送到刚才建的repo，推送完成删除本地分支gh-pages

找到新建的repo - setting-pages做如下修改

![logo](0.png ':size=WIDTHxHEIGHT')

修改 `vue.config.js`增加`publishPath`
```
const { defineConfig } = require('@vue/cli-service')
module.exports = defineConfig({
  transpileDependencies: true,
  lintOnSave: false,
  publicPath:'./',  //防止build后index.html打开空白
})

```

项目内安装push-dir用于推送dist目录至github page

```
npm install push-dir --save-dev
```

package.json 中添加一条新的脚本命令：
```
{
  "scripts": {
    "deploy": "push-dir --dir=dist --branch=gh-pages --cleanup"
  }
}
```

`npm run build`编译vue项目生成dist；成功后运行 `npm run deploy` 将dist推送至gh-pages

浏览打开repo -setting - pages 中的网址 就可以访问了。


