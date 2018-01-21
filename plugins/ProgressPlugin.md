# ProgressPlugin

文档无

```javascript
class ProgressPlugin {
  constructor(options) {
    if(typeof options === "function") { 
      options = {
        handler: options
      };
    }
    options = options || {};
    this.profile = options.profile; // 打印耗时
    this.handler = options.handler; // 自定义打印函数
  }

  apply(compiler) {
    const handler = this.handler || defaultHandler;
    const profile = this.profile;
    if(compiler.compilers) { /* ... */
    } else {

      //...
      
      compiler.plugin("emit", (compilation, callback) => {
        handler(0.95, "emitting");
        callback();
      });
      compiler.plugin("done", () => {
        handler(1, "");
      });
    }

    function defaultHandler(percentage, msg) {
      // ...
      // 打印编译进度
      percentage = Math.floor(percentage * 100);
      msg = `${percentage}% ${msg}`;
      process.stderr.write(msg);
      
      if(profile) { /* 打印各时间段耗时 */
      }
    }
  }
}
```
