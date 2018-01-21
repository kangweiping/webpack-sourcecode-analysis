#DefinePlugin
[英文文档](https://webpack.js.org/plugins/define-plugin/)
[中文文档](https://doc.webpack-china.org/plugins/define-plugin/)
```javascript
class DefinePlugin {
	constructor(definitions) {
		this.definitions = definitions;
	}
	apply(compiler) {
		const definitions = this.definitions;
		compiler.plugin("compilation", (compilation, params) => {
      // ...
			params.normalModuleFactory.plugin("parser", (parser) => {
				(function walkDefinitions(definitions, prefix) {
					Object.keys(definitions).forEach((key) => {
						const code = definitions[key];
						if(code && typeof code === "object" && !(code instanceof RegExp)) { // 正则表达式
							walkDefinitions(code, prefix + key + ".");
							applyObjectDefine(prefix + key, code);
							return;
						}
						applyDefineKey(prefix, key);
						applyDefine(prefix + key, code);
					});
				}(definitions, ""));

				function stringifyObj(obj) {
					return "Object({" + Object.keys(obj).map((key) => {
						const code = obj[key];
						return JSON.stringify(key) + ":" + toCode(code);
					}).join(",") + "})";
				}

        // 转换类型
				function toCode(code) {
					if(code === null) return "null";
					else if(code === undefined) return "undefined";
					else if(code instanceof RegExp && code.toString) return code.toString();
					else if(typeof code === "function" && code.toString) return "(" + code.toString() + ")";
					else if(typeof code === "object") return stringifyObj(code);
					else return code + "";
				}

        // ... 
        function applyDefine(key, code) { code = toCode(code); }
        // ... 
			});
		});
	}
}
```