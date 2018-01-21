# BannerPlugin 

[英文文档](https://webpack.js.org/plugins/banner-plugin/)
[中文文档](https://doc.webpack-china.org/plugins/banner-plugin/)

### 功能
为每个chunk文件头部添加banner

### 用法
```javascript
new webpack.BannerPlugin(banner)
// or
new webpack.BannerPlugin({
  banner: string, // 其值为字符串，将作为注释存在
  raw: boolean, // 如果值为 true，将直出，不会被作为注释
  entryOnly: boolean, // 如果值为 true，将只在入口 chunks 文件中添加
  test: string | RegExp | Array,
  include: string | RegExp | Array,
  exclude: string | RegExp | Array,
})
```

### 主要代码
```javascript
const ConcatSource = require("webpack-sources").ConcatSource;
const ModuleFilenameHelpers = require("./ModuleFilenameHelpers");

const wrapComment = (str) => {
  if(!str.includes("\n")) return `/*! ${str} */`;
  return `/*!\n * ${str.split("\n").join("\n * ")}\n */`;
};

class BannerPlugin {
  constructor(options) {
    if(arguments.length > 1)
      throw new Error("BannerPlugin only takes one argument (pass an options object)");
    if(typeof options === "string") // 用法1
      options = {
        banner: options
      };
    this.options = options || {};
    this.banner = this.options.raw ? options.banner : wrapComment(options.banner); // 如果options.raw值为true，将直出，否则用注释包裹
  }

  apply(compiler) {
    const options = this.options;
    const banner = this.banner;

    compiler.plugin("compilation", (compilation) => {
      compilation.plugin("optimize-chunk-assets", (chunks, callback) => {
        chunks.forEach((chunk) => {
          if(options.entryOnly && !chunk.isInitial()) return; // 如果options.entryOnly值为true，将只在入口 chunks 文件中添加
          chunk.files
            .filter(ModuleFilenameHelpers.matchObject.bind(undefined, options)) // 过滤文件 options.test,options.include,options.exclude
            .forEach((file) => {
              let basename;
              let query = "";
              let filename = file;
              const hash = compilation.hash;
              const querySplit = filename.indexOf("?");

              if(querySplit >= 0) {
                query = filename.substr(querySplit);
                filename = filename.substr(0, querySplit);
              }

              if(filename.indexOf("/") < 0) {
                basename = filename;
              } else {
                basename = filename.substr(filename.lastIndexOf("/") + 1);
              }

              /*
              * 对 banner 字符串中的占位符取值（替换代码在lib/TemplatedPathPlugin.js中）
              * new webpack.BannerPlugin({
              *   banner: "hash:[hash], chunkhash:[chunkhash], name:[name], filebase:[filebase], query:[query], file:[file]"
              * })
              */
              const comment = compilation.getPath(banner, { // options.banner
                hash,
                chunk,
                filename,
                basename,
                query,
              });
              
              // 拼接代码
              return compilation.assets[file] = new ConcatSource(comment, "\n", compilation.assets[file]);
            });
        });
        callback();
      });
    });
  }
}
```
```javascript
ModuleFilenameHelpers.matchPart = function matchPart(str, test) {
  if(!test) return true;
  test = asRegExp(test);
  if(Array.isArray(test)) {
    return test.map(asRegExp).filter(function(regExp) {
      return regExp.test(str);
    }).length > 0;
  } else {
    return test.test(str);
  }
};

ModuleFilenameHelpers.matchObject = function matchObject(obj, str) {
  if(obj.test) // options.test
    if(!ModuleFilenameHelpers.matchPart(str, obj.test)) return false;
  if(obj.include) // options.include
    if(!ModuleFilenameHelpers.matchPart(str, obj.include)) return false;
  if(obj.exclude) // options.exclude
    if(ModuleFilenameHelpers.matchPart(str, obj.exclude)) return false;
  return true;
};
```