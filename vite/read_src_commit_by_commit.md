## init	820c2cf	Evan You <yyx990803@gmail.com>	2020年4月20日 下午4:00

大致思路：监听本地文件、拦截请求、浏览器热更新

启动一个 server，通过 vue 相关中间件对于 .vue 文件的请求拦截，返回编译改造后的 vue 组件代码

监听本地 .vue 文件，如果有改动，通过 websocket 发送事件实现热更新等

页面需引入一个 js，用于创建 ws，监听 server 发送的一些事件，执行相应更新操作


vue 中间件：

对于一个 .vue 文件
```
<template>
  <div class="name">xx{{ name }}</div>
</template>

<script>
export default {
  data() {
    return { 
      name: 'peter'
    }
  }
}
</script>

```

通过 http://localhost:3000/demo.vue 后，会得到

```
const script = {
  data() {
    return { 
      name: 'peter'
    }
  }
}

export default script
import { render } from "/demo.vue?type=template"
script.render = render
script.__hmrId = "/demo.vue"
```

访问 /demo.vue?type=template 这种带有参数的连接，那就返回带有 render 方法的代码

```
...
export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", _hoisted_1, "xx" + _toDisplayString(_ctx.name), 1 /* TEXT */))
}
```

## feat: auto inject hmr client	4a04d81	Evan You <yyx990803@gmail.com>	2020年4月20日 下午4:45

`parseSFC` 方法默认不走缓存，考虑热更新的场景，如果走缓存就没办法拿到最新的

# feat: module rewrite	33488fe	Evan You <yyx990803@gmail.com>	2020年4月20日 下午5:32

变量名带上 __ 前缀

`moduleRewriter.js` 专门改造 sfc-compile 生成的代码:
`.vue` 返回的 script 不是简单的拼凑，通过 babel/parser 解析，找到 `import` 和 `export default`，对于 import npm 包的，统一改为 '/__modules/xx'

`moduleMiddleware.js` 就是返回 node_module 包的代码

## refactor: use async fs + expose createServer API	bfb4b91	Evan You <yyx990803@gmail.com>	2020年4月21日 上午6:14

读取文件 fs 改成异步

resp 返回文件内容改成 stream

模块查找基于 cwd 路径

对于 __modules/xx 请求 return moduleMiddleware(pathname.replace('/__modules/', ''), res)

## refactor: use TS	91d76bf	Evan You <yyx990803@gmail.com>	2020年4月21日 上午7:13

@npm
- `npm-run-all` 并行/串行 执行 npm script

ts 配置
```
{
  "compilerOptions": {
    "sourceMap": false,
    "target": "esnext",
    "moduleResolution": "node",
    
    "esModuleInterop": true, // 增加一些辅助代码，保证 import default 和 import * 对于 commonjs 代码不会出错
    
    "declaration": true, // 生成 d.ts
    "allowJs": false,
    
    "allowSyntheticDefaultImports": true, // false 则：对于没有 export default 的 xx.js ，那么 import x from xx.js 就会报错
    
    "noUnusedLocals": true,
    "strictNullChecks": true, // null和 undefined值不包含在任何类型里，只允许用它们自己和 any来赋值（有个例外， undefined可以赋值到 void）
    "noImplicitAny": true, // true: 有隐含的 any类型时报错
    "removeComments": false,
    "lib": [
      "esnext",
      "DOM"
    ]
  }
}
```
[typescript - Understanding esModuleInterop in tsconfig file - Stack Overflow](https://stackoverflow.com/questions/56238356/understanding-esmoduleinterop-in-tsconfig-file)

## add tests	e718bd5	Evan You <yyx990803@gmail.com>	2020年4月21日 上午8:57

@npm
- `execa` 是可以调用shell和本地外部程序的javascript封装。会启动子进程执行。支持多操作系统，包括windows。如果父进程退出，则生成的全部子进程都被杀死



`test/test.js`
```
server.kill('SIGTERM', {
  forceKillAfterTimeout: 2000
})
```
SIGTERM 程序结束(terminate)信号，与SIGKILL不同的是该信号可以被阻塞和处理。通常用来要求程序自己正常退出。shell命令kill缺省产生这个信号。SIGTERM is the default signal sent to a process by the kill or killall commands.

所以这里 forceKillAfterTimeout: 2000 就表示，如果 2 秒了， server 进程还没自己正常退出，那就强杀

整个测试的思路就是，准备一个项目
```
xx.vue
main.js
index.html
```
然后拷贝到临时目录，启动 server，用 puppeteer 做 e2e

## remove git add from lint-staged	3e64a74	Evan You <yyx990803@gmail.com>	2020年4月21日 上午8:58

## properly handle cwd	93286b9	Evan You <yyx990803@gmail.com>	2020年4月21日 上午10:04

@npm
- `resolve-from` Resolve the path of a module like require.resolve() but from a given path, 代替 `resolve-cwd`
- `minimist`轻量命令行参数解析器

### `moduleResolver.js`

增强了  module 查找能力

```
modulePath = resolve(cwd, `${id}/package.json`)
```

支持 `.map` soucemap 文件

### `server.js`

createServer 带上 cwd 参数，后续所有跟启动路径相关的地方都传入，这样别的地方就不存在 `process.cwd()`

## refactor types	f4382f1	Evan You <yyx990803@gmail.com>	2020年4月21日 上午11:08


`vueCompiler.js` 把 compile script 和 compile template 封装

## feat: style hot reload	140f2b2	Evan You <yyx990803@gmail.com>	2020年4月21日 下午12:37

@npm
- `hash-sum` blazing fast unique hash generator; 输出一个变量/function，计算 hash

### `watcher.js`

style hotreload 的处理：

descriptor.styles 前后两次比较，
- 如果有 `scoped` 改变，则发送 `reload`事件
- 顺序遍历比较， 如有不同发送 `style-update' 指定索引
- 新的更长，则旧的部分删除，发送 `style-remove` 指定 `{id: `${hash_sum(resourcePath)}-${i + nextStyles.length}`}`

比较两个 block 是不是一样:
- 强等于比较
- keys 长度比较
- 遍历 keys 比较属性

### `vueCompiler.js`

`compileSFCMain()`

对于有 scoped 的 style block，请求 `.vue` 的代码包含
```
import "xxx?type=style&index={i}&${timestap}"
```

`compileSFCTemplate()`

带上参数 id: `data-v-${id}`, 其中 ${id} 就是请求的 `pathname`

`compileSFCStyle()`

请求 type=style，就编译，拼接创建 style 标签的代码，标签 id 为 `vue-style-${id}-${index}`

## rename	be00e79	Evan You <yyx990803@gmail.com>	2020年4月21日 下午1:04

vds -> vite

## include bin	a47c406	Evan You <yyx990803@gmail.com>	2020年4月21日 下午1:06

package.json
```
  "files": [
    "bin",
    "dist"
  ],
```

指定 npm 包含哪些内容

## ci	c76ca14	Evan You <yyx990803@gmail.com>	2020年4月21日 下午1:21

顺手学了下 circle workflow

`yarn --frozen-lockfile` 表示不生成/修改 yarn.lock

## test: fix puppeteer on ci	97de06e	Evan You <yyx990803@gmail.com>	2020年4月21日 下午1:47

[puppeteer/troubleshooting.md at main · puppeteer/puppeteer](https://github.com/puppeteer/puppeteer/blob/main/docs/troubleshooting.md#setting-up-chrome-linux-sandbox)


## fix: use correct vue & compiler-sfc	0d5a2a4	Evan You <yyx990803@gmail.com>	2020年4月21日 下午11:39

vue compiler 也需要从 cwd 用户本地去找，而不是直接从 vite 依赖的 npm 里面加载

## refactor: use koa	c74b24e	Evan You <yyx990803@gmail.com>	2020年4月22日 上午5:01

## use custom history fallback	a307eeb	Evan You <yyx990803@gmail.com>	2020年4月22日 上午5:32

## feat: support import rewriting in index.html	4ed433a	Evan You <yyx990803@gmail.com>	2020年4月22日 上午6:06

支持 index.html 中
```
<script>
import xx from xxx
<script>
```

![image](https://user-images.githubusercontent.com/4012276/88315228-9e931480-cd48-11ea-9950-cf16cebcca09.png)

## refactor: use es-module-lexer	5780a2a	Evan You <yyx990803@gmail.com>	2020年4月22日 下午1:32

@npm
- `es-module-lexer` Outputs the list of exports and locations of import specifiers, including dynamic import and import meta handling.

`es-module-lexer` 通过编译成 native 代码提速，也比 babel 轻量一点，vite 只需要 import 部分的代码

## add some caching	ef95a00	Evan You <yyx990803@gmail.com>	2020年4月22日 下午3:09

@npm
- `koa-conditional-get` `koa-etag` 一起使用，通过 ETag 响应和 If-None-Match 请求做 HTTP 缓存

读文件流加上缓存，代码实现 lastModified

## more aggressive vue compilation caching	f6ef1b1	Evan You <yyx990803@gmail.com>	2020年4月22日 下午4:04

vue 编译缓存，检测到文件修改，移除 vue 编译相应缓存

## feat: hmr propagation	6e66766	Evan You <yyx990803@gmail.com>	2020年4月23日 上午9:30

rewriteImport 作为入口，建立两个map，分别是 <importer, importee> 和 <importee， importer>，方便后面一个 importee 修改了，递归对应 importer 来判断是否要 full reload，目前如果往上递归找到一个没有被依赖的资源，那么 full reload，可以理解成，如果一个依赖树被砍掉一部分，则 full resolve


## feat: support resolving snowpack web_modules (#4)	a183791	Israel Roldan <israel@palu.io>	2020年4月23日 下午9:50

snowpack 是将 npm 包映射安装到 web_modules/ 下,
@TODO 暂时还没懂支持 snowpack 是怎么回事

## refactor: use /@ for special requests	ae3c83a	Evan You <yyx990803@gmail.com>	2020年4月23日 下午10:01

`isHotBoundary` 目前的热更新递归边界就是到一个 **非 .vue** （的js）为止

`hrm.js` 里面的一大段解释：

了解下 webpack 里面热更新相关的 accept 是什么用途

[模块热替换 | webpack](https://webpack.docschina.org/guides/hot-module-replacement/)

例如 index.js 里面 import a.js，那么就在 index.js 里面去 accept(a.js)，因为当 a.js 修改了，a.js 本身是能通知到更新的，那么其它 importer 就需要去声明监听 a.js 的变化

### HMR 工作原理

1. `vue` 文件响应前，会转成 `js` 文件
2. 所有`.js`文件在响应前，都会经过解析，以便重写最终 import 的模块路径。在这个解析过程，顺便记录了 importer/impotee 的映射关系，用于 HMR 分析
3. 当一个 `.vue` 文件修改了，直接读取解析，然后通知浏览器，这里 `.vue` 文件本身就能 accept 自己的变化
4. 当一个 `.js` 文件变化，会触发 HMR 模块关系分析，遍历它的 importer 链，试图找到一个 HMR 边界，这个边界的定义就是看它，有没有声明了 accept，通常 import `@hmr` 的就是声明了 accept。可以这么理解吧，例如 a import b, b import c，当 c 修改，b 成为边界之后，此时先不用考虑 a，因为 a 后续也会成为 b 的边界
5. 如果 importer 链上不存在一个 HMR boundary，就认为这个模块 "dead end"，你可以理解为，没有模块在意它的变化，那就是死掉了
6. 如果是 `.vue` boundary，添加到 `vueImports` 集合
7. 如果是 `.js` boundary，检查它现在的 child importer 是否在这个 boundary 的 accept 列表中，如果是的话，把 child importer 添加到 `jsImporters` 集合
8. 如果遍历完之后，没有 dead end，通知浏览器更新所有 `jsImporters` 和 `vueImporters`

如何获得 accepted 列表：
- `import rewrite` 的 `js` 文件的时候，如果存在 `/@hmr` import，那么会完整解析文件，找到 `hot.accept` 调用，然后记录它与 accpeted 依赖关系，保存在 `hmrBoundariesMap`
- boundary 文件的完整路径也会注入到 `hot.accept` 相关调用代码，后续就可以用完整路径拿到依赖的完整路径

### 代码梳理

`client.js`

```
 export const hot = {
   accept(
    importer: string,
    deps: string | string[],
    callback: (modules: object | object[]) 
   ) {
```

相当于在浏览器注册了 importer 和 deps 的关系，当

@TODO

## feat: js hmr API	3e5076d	Evan You <yyx990803@gmail.com>	2020年4月23日 下午11:16

## feat: vite build	0ea7970	Evan You <yyx990803@gmail.com>	2020年4月24日 上午11:04

## chore: setup changelog	2dba0d0	Evan You <yyx990803@gmail.com>	2020年4月27日 下午11:04

@npm 
- `conventional-changelog-cli`

## v0.8.0	4ce94b6	Evan You <yyx990803@gmail.com>	2020年5月1日 上午12:18



