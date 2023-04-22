# npm or yarn or pnpm

## npm
`npm init -y`生成package.json
`npm intall xxx`安装依赖包，但是依赖包中如果有其他依赖会存在依赖包中嵌套依赖包，这样会导致依赖结构复杂，同样的依赖包重复，占用较大磁盘空间。致命问题：windows文件路径最长260多字节，嵌套会长过windows路径的长度限制。

node_modules
├── A@1.0.0
│   └── node_modules
│       └── B@1.0.0
├── C@1.0.0
│   └── node_modules
│       └── B@2.0.0
└── D@1.0.0
    └── node_modules
        └── B@1.0.0

## npm3
为了解决嵌套过长问题，npm3会将依赖铺平，放到同一层，从而解决了路径长度问题。（仍然存在嵌套，是因为一个包有多个版本，提升只能提升一个，所以后面在遇到相同的包不同版本将会是嵌套）

node_modules
├── A@1.0.0
├── D@1.0.0
├── B@1.0.0
└── C@1.0.0
    └── node_modules
        └── B@2.0.0

**npm3遗留问题**
1. 幽灵依赖：指在 `package.json` 中未定义的依赖，但项目中依然可以正确地被引用到。如只安装了A和C，由于A依赖于B，所以会将B提升至与A同级，且项目引用B时正常工作的。如果某天某个版本的A依赖不在依赖B或者B的版本发生变化，那么就会造成依赖缺失或兼容性问题。
2. 不确定性：同样的`package.json`文件，`install`依赖后可能不会得到同样的`node_modules`目录结构。比如A依赖B@1.0，C依赖B@2.0，依赖安装后究竟应该提升1.0还是2.0。取决于用户的安装顺序，先安装的提升。但是如果删除本地`node_modules`然后再重新`install`，安装顺序会按照`package.json`依赖顺序，可能会导致`node_modules`目录结构不同，代码无法正常运行。
3. 依赖分身：如果没有被提升的依赖版本在多个依赖中引用，还是会有重复安装的问题。
- A和D依赖B@1.0
- C和E依赖B@2.0

node_modules
├── A@1.0.0
├── B@1.0.0
├── D@1.0.0
├── C@1.0.0
│   └── node_modules
│       └── B@2.0.0
└── E@1.0.0
    └── node_modules
        └── B@2.0.0

## yarn
`yarn`也采用了扁平化的`node_modules`结构，它解决了`npm3`迫在眉睫的问题：**依赖安装慢，不确定性**。
**提升安装速度：**
在`npm`中安装依赖，任务是`串行`的，会按包的顺序逐个执行安装，这意味着它会等待一个包完全安装才会继续下一个。
为了加快安装速度，yarn采用了`并行`安装，在性能上有显著提高。而且在缓存机制上，yarn会将每个包缓存在磁盘上，在下一次安装这个包，可以从磁盘离线安装。
**解决不确定性：**
`yarn.lock`:安装依赖会根据`package.json`生成一份`yarn.lock`文件。它记录了依赖以及依赖的子依赖，依赖的版本，获取地址与验证模块完整性的`hash`。即使不同的安装顺序，相同的依赖关系在任何环境容器中都能得到稳定的`node_modules`目录结构，从而保证了确定性。
> PS：`npm5`同样也发布了`package-lock.json`

#### `package-lock.json`作用
```json
"@babel/code-frame": {
  "version": "7.21.4",
  "resolved": "https://registry.npmmirror.com/@babel/code-frame/-/code-frame-7.21.4.tgz",
  "integrity": "sha512-LYvhNKfwWSPpocw8GI7gpK2nq3HSDuEPC/uSYaALSJu9xjsalaaYFOq0Pwt5KmVqwEbZlDu81aLXwBOmD/Fv9g==",
  "requires": {
    "@babel/highlight": "^7.18.6"
  }
}
```
包含每个模块对应的版本，压缩包地址，依赖和完整`hash`
- 锁定安装模块版本号
- 固定依赖版本位置，避免项目结构混乱
- 完整`hash`是用来验证资源的完整性，防止别人篡改内容

## pnpm
策略：内容寻址存储`CAS Content-addressable store`
该策略会将包安装在系统的`全局store`中，依赖的每个版本只会在系统中安装一次。
硬链接（`Hard link`）：源文件副本，它指向的`inode`和源文件指向的`inode`是同一个，包含文件的一些基本信息。它使得用户可以通过路径引用查找到全局`store`中的源文件，而且这个副本不占任何空间。同时，`pnpm`会在`全局store`里存储硬链接，不同项目可以从`全局store`找到同一个依赖，大大节省了磁盘空间。
软链接（`Synmbolic link`）：可以理解为快捷方式，它指向的`inode`是源文件地址，`pnpm`可以通过它找到对应磁盘目录下的依赖地址。`pnpm`包和包之间的依赖关系是通过软链接组织的。


node_modules
├── .pnpm
│   ├── A@1.0.0
│   │   └── node_modules
│   │       ├── A => <store>/A@1.0.0 硬链接
│   │       └── B => ../../B@1.0.0 软链接
│   ├── B@1.0.0
│   │   └── node_modules
│   │       └── B => <store>/B@1.0.0
│   ├── B@2.0.0
│   │   └── node_modules
│   │       └── B => <store>/B@2.0.0
│   └── C@1.0.0
│       └── node_modules
│           ├── C => <store>/C@1.0.0
│           └── B => ../../B@2.0.0
│
├── A => .pnpm/A@1.0.0/node_modules/A 软连接（指向pnpm中的地址）
└── C => .pnpm/C@1.0.0/node_modules/C