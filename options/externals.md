# externals

### 功能
外部扩展,使我们可以全局引用js文件，但任然可以使用模块化的方式进行代码编写
例如
import Vue from 'vue';
正常情况下会把vue的源码打包到最后生成的bundle中,
但如果配置了externals: {vue: 'Vue'}，则不会打包vue.js源码，生成的内容会变成
module.exports = Vue

### 用法
```javascript
{
  externals: [String|Array|RegExp|Function|Object]
}
{
  externals: {
    vue: 'Vue',
    react: 'React',
    jquery: 'jQuery'
  }
}
```

### 主要代码
```javascript
// WebpackOptionsApply.js process(options) 使用externals配置
if(options.externals) {
  ExternalsPlugin = require("./ExternalsPlugin");
  compiler.apply(new ExternalsPlugin(options.output.libraryTarget, options.externals));
}

// ExternalsPlugin.js 注册plugin到normalModuleFactory
class ExternalsPlugin {
  constructor(type, externals) {
    this.type = type;
    this.externals = externals;
  }
  apply(compiler) {
    compiler.plugin("compile", (params) => {
      params.normalModuleFactory.apply(new ExternalModuleFactoryPlugin(this.type, this.externals));
    });
  }
}

// ExternalModuleFactoryPlugin.js 处理各种类型参数，匹配后生成ExternalModule
class ExternalModuleFactoryPlugin {
  constructor(type, externals) {
    this.type = type;
    this.externals = externals;
  }

  apply(normalModuleFactory) {
    const globalType = this.type;
    // factory是正常流程下的创建module方法
    normalModuleFactory.plugin("factory", factory => (data, callback) => {
      const context = data.context;
      const dependency = data.dependencies[0]; // import vue form 'vue' => 'vue'即是dependency.request

      /*
      * 最后生成模块代码为module.exports = value;
      * type决定最后代码的生成类型
      */
      function handleExternal(value, type, callback) {
        if(typeof type === "function") {
          callback = type;
          type = undefined;
        }
        if(value === false) return factory(data, callback); // 不匹配的话，走正常流程 NormalModuleFactory下的factory
        if(value === true) value = dependency.request;
        if(typeof type === "undefined" && /^[a-z0-9]+ /.test(value)) { 
          const idx = value.indexOf(" ");
          type = value.substr(0, idx);
          value = value.substr(idx + 1);
        }
        callback(null, new ExternalModule(value, type || globalType, dependency.request));
        return true;
      }
      (function handleExternals(externals, callback) {
        if(typeof externals === "string") { // 字符串
          if(externals === dependency.request) {
            return handleExternal(dependency.request, callback);
          }
        } else if(Array.isArray(externals)) { // 数组，不知道为啥要这样用
          let i = 0;
          (function next() {
            let asyncFlag;
            const handleExternalsAndCallback = function handleExternalsAndCallback(err, module) {
              if(err) return callback(err);
              if(!module) {
                if(asyncFlag) {
                  asyncFlag = false;
                  return;
                }
                return next();
              }
              callback(null, module);
            };

            do {
              asyncFlag = true;
              if(i >= externals.length) return callback();
              handleExternals(externals[i++], handleExternalsAndCallback);
            } while (!asyncFlag); // eslint-disable-line keyword-spacing
            asyncFlag = false;
          }());
          return;
        } else if(externals instanceof RegExp) { // 正则表达式
          if(externals.test(dependency.request)) {
            return handleExternal(dependency.request, callback);
          }
        } else if(typeof externals === "function") { // 函数
          externals.call(null, context, dependency.request, function(err, value, type) {
            if(err) return callback(err);
            if(typeof value !== "undefined") {
              handleExternal(value, type, callback);
            } else {
              callback();
            }
          });
          return;
        } else if(typeof externals === "object" && Object.prototype.hasOwnProperty.call(externals, dependency.request)) { // 对象
          return handleExternal(externals[dependency.request], callback);
        }
        callback();
      }(this.externals, function(err, module) {
        if(err) return callback(err);
        if(!module) return handleExternal(false, callback);
        return callback(null, module);
      }));
    });
  }
}

/*
* ExternalModule.js build Module, 生成Module代码
* NormalModule和ExternalModule
* 正常情况下生成NormalModule，如果配置了options.externals并且匹配，则会生成ExternalModule
*/
class ExternalModule extends Module {
  constructor(request, type, userRequest) {
    super();
    // ...
  }
  // ...

  // 什么都不干，如果是NormalModule则会进行读取模块文件，使用loader处理文件内容等操作
  build(options, compilation, resolver, fs, callback) {
    this.builtTime = Date.now();
    callback();
  }
  
  // 最后默认生成的Module内容
  getSourceForDefaultCase(optional, request) {
    const missingModuleError = optional ? this.checkExternalVariable(request, request) : "";
    return `${missingModuleError}module.exports = ${request};`;
  }

  getSourceString() {
    // ...
    return this.getSourceForDefaultCase(this.optional, request);
    }
  }

  getSource(sourceString) {
    return new RawSource(sourceString);
  }

  source() {
    return this.getSource(
      this.getSourceString()
    );
  }
}
```