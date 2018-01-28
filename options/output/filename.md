# output.filename,output.chunkFilename,output.publicPath

### 功能
输出文件名

### 用法
asset-path

```javascript
{
  output: {
    filename: '[name]/[name].[chunkhash].js',
    chunkFilename: '[id].[chunkhash].js',
    publicPath: '../../'
  }
}
```

### 主要代码
```javascript
// Compilation.js
getPath(filename, data) {
  data = data || {};
  data.hash = data.hash || this.hash;
  return this.mainTemplate.applyPluginsWaterfall("asset-path", filename, data); // asset-path
}
createChunkAssets() { // 创建chunk文件
  const outputOptions = this.outputOptions;
  const filename = outputOptions.filename;
  const chunkFilename = outputOptions.chunkFilename;
  for(let i = 0; i < this.chunks.length; i++) {
    const chunk = this.chunks[i];
    const filenameTemplate = chunk.filenameTemplate ? chunk.filenameTemplate :
        chunk.isInitial() ? filename :
        chunkFilename;
    // ...  
    file = this.getPath(filenameTemplate, { // 获取文件名
      noChunkHash: !useChunkHash,
      chunk
    });

    this.assets[file] = source;
  }
}

// TemplatedPathPlugin.js 
// 占位符[hash, chunkhash, name, id, file, query, filebase]
const REGEXP_HASH = /\[hash(?::(\d+))?\]/gi,
  REGEXP_CHUNKHASH = /\[chunkhash(?::(\d+))?\]/gi,
  REGEXP_NAME = /\[name\]/gi,
  REGEXP_ID = /\[id\]/gi,
  REGEXP_FILE = /\[file\]/gi,
  REGEXP_QUERY = /\[query\]/gi,
  REGEXP_FILEBASE = /\[filebase\]/gi;
  
const replacePathVariables = (path, data) => { // 使用实际的值替换占位符
  const chunk = data.chunk;
  const chunkId = chunk && chunk.id;
  const chunkName = chunk && (chunk.name || chunk.id);
  const chunkHash = chunk && (chunk.renderedHash || chunk.hash);
  const chunkHashWithLength = chunk && chunk.hashWithLength;

  if(data.noChunkHash && REGEXP_CHUNKHASH_FOR_TEST.test(path)) {
    throw new Error(`Cannot use [chunkhash] for chunk in '${path}' (use [hash] instead)`);
  }

  return path
    .replace(REGEXP_HASH, withHashLength(getReplacer(data.hash), data.hashWithLength))
    .replace(REGEXP_CHUNKHASH, withHashLength(getReplacer(chunkHash), chunkHashWithLength))
    .replace(REGEXP_ID, getReplacer(chunkId))
    .replace(REGEXP_NAME, getReplacer(chunkName))
    .replace(REGEXP_FILE, getReplacer(data.filename))
    .replace(REGEXP_FILEBASE, getReplacer(data.basename))
    // query is optional, it's OK if it's in a path but there's nothing to replace it with
    .replace(REGEXP_QUERY, getReplacer(data.query, true));
};

class TemplatedPathPlugin {
  apply(compiler) {
    compiler.plugin("compilation", compilation => {
      const mainTemplate = compilation.mainTemplate;

      mainTemplate.plugin("asset-path", replacePathVariables); // 注册plugin

      // ...
    });
  }
}
```