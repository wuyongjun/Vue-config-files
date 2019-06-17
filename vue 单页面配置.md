##### 多页面搭建

###### 基于cli2搭建

- npm init webpack <progrom>
- cd <progrom>

###### 配置文件修改

1.安装 glob 

```
$ npm install glob --save-dev
```
并且调整 项目目录

```
src/
	pages/
		index/
			index.js
			App.js
			index.html
		home/
			home.js
			home.html
			Home.vue
```

   原来的多入口改为单入口
2.修改build下面的  utils.js 

```
var path = require('path')
var config = require('../config')
var ExtractTextPlugin = require('extract-text-webpack-plugin')

exports.assetsPath = function (_path) {
    var assetsSubDirectory = process.env.NODE_ENV === 'production' ?
        config.build.assetsSubDirectory :
        config.dev.assetsSubDirectory
    return path.posix.join(assetsSubDirectory, _path)
}

exports.cssLoaders = function (options) {
    options = options || {}

    var cssLoader = {
        loader: 'css-loader',
        options: {
            minimize: process.env.NODE_ENV === 'production',
            sourceMap: options.sourceMap
        }
    }

    // generate loader string to be used with extract text plugin
    function generateLoaders(loader, loaderOptions) {
        var loaders = [cssLoader]
        if (loader) {
            loaders.push({
                loader: loader + '-loader',
                options: Object.assign({}, loaderOptions, {
                    sourceMap: options.sourceMap
                })
            })
        }

        // Extract CSS when that option is specified
        // (which is the case during production build)
        if (options.extract) {
            return ExtractTextPlugin.extract({
                use: loaders,
                fallback: 'vue-style-loader'
            })
        } else {
            return ['vue-style-loader'].concat(loaders)
        }
    }

    // https://vue-loader.vuejs.org/en/configurations/extract-css.html
    return {
        css: generateLoaders(),
        postcss: generateLoaders(),
        less: generateLoaders('less'),
        sass: generateLoaders('sass', { indentedSyntax: true }),
        scss: generateLoaders('sass'),
        stylus: generateLoaders('stylus'),
        styl: generateLoaders('stylus')
    }
}

// Generate loaders for standalone style files (outside of .vue)
exports.styleLoaders = function (options) {
    var output = []
    var loaders = exports.cssLoaders(options)
    for (var extension in loaders) {
        var loader = loaders[extension]
        output.push({
            test: new RegExp('\\.' + extension + '$'),
            use: loader
        })
    }
    return output
}

/* 这里是添加的部分 ---------------------------- 开始 */

// glob是webpack安装时依赖的一个第三方模块，还模块允许你使用 *等符号, 例如lib/*.js就是获取lib文件夹下的所有js后缀名的文件
var glob = require('glob')
// 页面模板
var HtmlWebpackPlugin = require('html-webpack-plugin')
// 取得相应的页面路径，因为之前的配置，所以是src文件夹下的pages文件夹
var PAGE_PATH = path.resolve(__dirname, '../src/pages')
// 用于做相应的merge处理
var merge = require('webpack-merge')


//多入口配置
// 通过glob模块读取pages文件夹下的所有对应文件夹下的js后缀文件，如果该文件存在
// 那么就作为入口处理
exports.entries = function () {
    var entryFiles = glob.sync(PAGE_PATH + '/*/*.js')
    var map = {}
    entryFiles.forEach((filePath) => {
        var filename = filePath.substring(filePath.lastIndexOf('\/') + 1, filePath.lastIndexOf('.'))
        map[filename] = filePath
    })
    return map
}

//多页面输出配置
// 与上面的多页面入口配置相同，读取pages文件夹下的对应的html后缀文件，然后放入数组中
exports.htmlPlugin = function () {
    let entryHtml = glob.sync(PAGE_PATH + '/*/*.html')
    let arr = []
    entryHtml.forEach((filePath) => {
        let filename = filePath.substring(filePath.lastIndexOf('\/') + 1, filePath.lastIndexOf('.'))
        let conf = {
            // 模板来源
            template: filePath,
            // 文件名称
            filename: filename + '.html',
            // 页面模板需要加对应的js脚本，如果不加这行则每个页面都会引入所有的js脚本
            chunks: ['manifest', 'vendor', filename],
            inject: true
        }
        if (process.env.NODE_ENV === 'production') {
            conf = merge(conf, {
                minify: {
                    removeComments: true,
                    collapseWhitespace: true,
                    removeAttributeQuotes: true
                },
                chunksSortMode: 'dependency'
            })
        }
        arr.push(new HtmlWebpackPlugin(conf))
    })
    return arr
}
/* 这里是添加的部分 ---------------------------- 结束 */
```

3.  入口配置   build  下面的 webpack.base.config.js

   ```
   	...
   module.exports = {
     context: path.resolve(__dirname, '../'),
     // entry: {
     //   app: './src/main.js'
     // },
     entry: utils.entries(), //添加的
     output: {
     ...
   ```

   

4. 开发配置  webpack.dev.config.js

   ```
    .....
    new webpack.NoEmitOnErrorsPlugin(),
       // https://github.com/ampedandwired/html-webpack-plugin
       /* 注释这个区域的文件 ------------- 开始 */
       // new HtmlWebpackPlugin({
       //   filename: 'index.html',
       //   template: 'index.html',
       //   inject: true
       // }),
       /* 注释这个区域的文件 ------------- 结束 */
       new FriendlyErrorsPlugin()
   
       /* 添加 .concat(utils.htmlPlugin()) ------------------ */
     ].concat(utils.htmlPlugin())
     ....
   ```

   5.生产配置  webpack.prod.config.js

   ```
    	....
    /* 注释这个区域的内容 ---------------------- 开始 */
       // new HtmlWebpackPlugin({
       //   filename: config.build.index,
       //   template: 'index.html',
       //   inject: true,
       //   minify: {
       //     removeComments: true,
       //     collapseWhitespace: true,
       //     removeAttributeQuotes: true
       //     // more options:
       //     // https://github.com/kangax/html-minifier#options-quick-reference
       //   },
       //   // necessary to consistently work with multiple chunks via CommonsChunkPlugin
       //   chunksSortMode: 'dependency'
       // }),
       /* 注释这个区域的内容 ---------------------- 结束 */
   
       // split vendor js into its own file
       new webpack.optimize.CommonsChunkPlugin({
         name: 'vendor',
         minChunks: function (module, count) {
           // any required modules inside node_modules are extracted to vendor
           return (
             module.resource &&
             /\.js$/.test(module.resource) &&
             module.resource.indexOf(
               path.join(__dirname, '../node_modules')
             ) === 0
           )
         }
       }),
       // extract webpack runtime and module manifest to its own file in order to
       // prevent vendor hash from being updated whenever app bundle is updated
       new webpack.optimize.CommonsChunkPlugin({
         name: 'manifest',
         chunks: ['vendor']
       }),
       // copy custom static assets
       new CopyWebpackPlugin([{
         from: path.resolve(__dirname, '../static'),
         to: config.build.assetsSubDirectory,
         ignore: ['.*']
       }])
       /* 该位置添加 .concat(utils.htmlPlugin()) ------------------- */
     ].concat(utils.htmlPlugin())
   })
   
   if (config.build.productionGzip) {
   ....
   ```

   6.文件修改 

   如 home.js 

   ```
   import Vue from 'Vue'
   import Home from './Home.vue'
   
   /* eslint-disable no-new */
   new Vue({
     el: '#app',
     render: h => h(Home)
   })
   ```

   7. 修改config文件 index.js  打包好的文件使用相对路径

   ```
   buid:{
   	...
   	assetsPublicPath: '/',
   ```

	..
	// 修改为 
	..
	assetsPublicPath: './',
	...
   



##### cli3 搭建多页面

###### 新建项目  

```
$ npm uninstall vue-cli -g  // 如果有cli2
$ npm install @vue/cli -g 
$ vue create <proname>
```



###### 调整目录结构

 新建 src 下新建 views 目录 独立的多页面

```
src/
	view/
		index
			App.js
			main.js
		home
			App.js
			main.js
```

###### 根目录下新建 vue.config.js

```
module.exports = {
 publicPath: './',
 pages: {
     index: {
         entry:'src/views/index/main.js',
         template:'public/index.html',
         filename: 'index.html',
         title: 'index page'
     },
     home: {
        entry:'src/views/home/main.js',
        template:'public/index.html',
        filename: 'home.html',
        title: 'home page'
    }
 },
 // 是否生成sourceMap文件
 // 开发环境配置true，方便快速定位错误（APICloud控制台输出真的很难受）
 // 生产环境配置false，构建速度更快，打包之后体积更小
 productionSourceMap: true
}
```

###### 优化配置待续。。。。