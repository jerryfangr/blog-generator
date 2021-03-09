---
title: 一个简易的json数据库
date: 2021-03-05 20:41:23
tags: ['学习笔记', 'web', 'javascript']
---



## 开始

**本篇需要[node](https://nodejs.org/en/)环境**

**写测试用后台的时候，经常要增删改，用mock又感觉差点意思**

**干脆写一个可以链式调用的微型json数据库，JavaScript还能原生反序列化**

<br>



## 功能设计(假定数据库类叫JsonDatabase)

### 数据集合

* 创建集合链接

  ```javascript
  new JsonDatabase('dbname');
  ```

<br>

### 数据获取

* 获取所有数据

  ```javascript
  new JsonDatabase('dbname')
  	// data: []
  	.all()
  	// data: [1, 2, 3, 4, 5]
  	.get(dataList => {
  		// [1, 2, 3, 4, 5]
  	}, error => {
  		// error
  	})
  ```

* 数据分页

  ```javascript
  new JsonDatabase('dbname')
  	// data: []
  	.all()
  	// data: [1, 2, 3, 4, 5]
  	.limit(2, 3) // startIndex, numberOfData
  	// data: [3, 4, 5]
  	.get(dataList => {
  		// [2, 3, 4]
  	}, error => {
  		// error
  	})
  ```

* 数据过滤

  ```javascript
  new JsonDatabase('dbname')
  	// data: []
  	.all()
  	// data: [{id: 0, value: 'a1'}, {id: 1, value: 'a2'}, {id: 0, value: 'b'}]
  	.filter({value: /a\d+$/i})
  	// data: [{id: 0, value: 'a1'}, {id: 1, value: 'a2'}]
  	.get(dataList => {
  		// [{id: 0, value: 'a1'}, {id: 1, value: 'a2'}]
  	}, error => {
  		// error
  	})
  ```

<br>

### 数据修改

* 根据id补丁式修改

  ```javascript
  new JsonDatabase('dbname')
  	// data: []
  	.all()
  	// data: [{id: 0, value: 'a1'}, {id: 1, value: 'a2'}]
  	.update({
  		id: 0,
  		value: 'new a1',
  		time: 'new time'
  	})
  	// data: [{id: 0, value: 'new a1', time: 'new time'}, {id: 1, value: 'a2'}]
  	.get(dataList => {
  		// [{id: 0, value: 'new a1', time: 'new time'}, {id: 1, value: 'a2'}]
  	}, error => {
  		// error
  	})
  	// data: [{id: 0, value: 'new a1', time: 'new time'}, {id: 1, value: 'a2'}]
  	.save()
  ```

<br>

### 数据增加

* 自动创建id

  ```javascript
  new JsonDatabase('dbname')
  	// data: []
  	.all()
  	// data: []
  	.add(1)
  	// data: [{id: 0, value: 1}]
  	.add({name: 'name1'})
  	// data: [{id: 0, value: 1}, {id: 1, name: 'name1'}]
  	.save()
  ```

<br>

### 数据删除

* 根据id删除

  ```javascript
  new JsonDatabase('dbname')
  	.all()
  	// data: [{id: 0, name: 'a1'}, {id: 1, name: 'a2'}, {id: 2, name: 'a3'}]
  	.remove(1)
  	// data: [{id: 0, name: 'a1'}, {id: 2, name: 'a3'}]
  	.save()
  ```

<br>



## 代码实现

* cache.js

  ```javascript
  class CacheDatabase {
      static getInstance(dbName, data) {
          this.instance = this.instance || {};
          this.instance[dbName] = this.instance[dbName] || new this(data);
          return this.instance[dbName];
      }
  
      constructor(data) {
          this._data = data || null;
      }
  
      set data(data) {
          this._data = data;
      }
  
      get data() {
          return this._data;
      }
  
      clear() {
          delete CacheDatabase.instance[dbName];
          this._data = null;
      }
  }
  
  module.exports = CacheDatabase;
  ```

* database,js

  ```javascript
  const path = require('path');
  const CacheDatabase = require('./cache');
  const fs = require('fs');
  
  /**
   * * an json database
   * * it should start width [all()]r
   */
  class JsonDatabase {
      constructor(databaseName, databaseFilePath) {
          this._state = 'init';
          this._cacheDB = CacheDatabase.getInstance(databaseName);
          this._dbPath = databaseFilePath || path.join('db/json', databaseName + '.json');
          this._stack = Promise.resolve([]);
      }
  
      toString() {
          let status = 'db file: ' + this._dbPath;
          status += '\nstate: ' + this._state;
          return status;
      }
  
      /**
       * * read file content and convert to string;
       */  
      _readContent() {
          return new Promise((resolve, reject) => {
              fs.readFile(this._dbPath, function (err, data) {
                  if (err) { reject(err) };
                  resolve(data?.toString?.())
              });
          }).then((data) => {
              return JSON.parse(data || '[]')
          }, error => {
              if (error.errno === -4058) {
                  return [];
              }
              throw error;
          });
      }
  
      /**
       * * wirte content to file
       * @param {string} content 
       */
      _writeContent(content) {
          return new Promise((resolve, reject) => {
              fs.writeFile(this._dbPath, content, { flag: 'w', encoding: 'utf-8' }, function (err) {
                  if (err) { reject(err); }
                  resolve()
              });
          });
      }
  
      /**
       * * create a deep copy data
       * @param {*} data json format
       * @returns 
       */
      copy(data) {
          return JSON.parse(JSON.stringify(data || '[]'))
      }
  
      /**
     * * fetch all data
     * @param {allowModified} true: edit _cacheDB.data, false: edit copy of _cacheDB.data
     */
      all(allowModified) {
          this._state = 'select';
  
          if (this._cacheDB.data) {
              let data = allowModified ? this._cacheDB.data : this.copy(this._cacheDB.data);
              this._stack = Promise.resolve(data);
          } else {
              this._stack = this._readContent().then(data => {
                  this._cacheDB.data = allowModified ? data : this.copy(data);
                  return Promise.resolve(data);
              })
          }
          return this;
      }
  
      /**
       * * limit data 
       * @param {number} startIndex start index
       * @param {number} number number of data
       */
      limit(startIndex, number) {
          if (this._state === 'init') {
              throw new Error('Not allowed to start width limit funtion')
          }
          this._stack = this._stack.then(allDatas => {
              const endIndex = startIndex + (number || allDatas.length) - 1;
              const result = [];
              if (startIndex >= (allDatas.length - 1)) { return result; }
  
              for (let index = startIndex; index <= endIndex && allDatas[index]; index++) {
                  result.push(allDatas[index]);
              }
              return result;
          });
          return this;
      }
  
      /**
       * * filter data
       * @param {Object} options ex: { name: 'name',  age: /\d{1,2}/}
       */
      filter(options) {
          if (this._state === 'init') {
              throw new Error('Not allowed to start width filter funtion')
          }
          if (options.id) { options.id = parseInt(options.id, 10); }
          this._stack = this._stack.then(allDatas => {
              return allDatas.filter(d => {
                  for (const key in options) {
                      if (options[key] instanceof RegExp) {
                          if (!options[key].test(d[key])) {
                              return false;
                          }
                      } else if (options[key] !== d[key]) {
                          return false;
                      }
                  }
                  return true;
              });
          });
          return this;
      }
  
      /**
       * * get data by callback
       * @param {Function} success
       * @param {Function} fail
       */
      get(success, fail) {
          this._stack.then(data => {
              success(data);
              return data;
          }, fail);
          return this;
      }
  
      /**
       * * edit data
       * @param {Function} action edit function
       * @param {Boolean} force force do the action whitout select
       */
      _edit(action, force) {
          if (this._state !== 'edit' && !force && this._state !== 'select') {
              throw new Error('You have to select some data [all()], or set param force true');
          }
          this._state = 'edit';
          this._stack = this._stack.then(allDatas => {
              return action(allDatas);
          });
  
          return this;
      }
  
      /**
       * * add data
       * @param {Object} data
       * @param {Boolean} force force do this function
       */
      add(data, force) {
          return this._edit(allDatas => {
              const addValue = {};
              if (typeof data !== 'object' || Array.isArray(data)) {
                  addValue.value = data;
              } else {
                  addValue = data;
              }
  
              const lastData = allDatas[allDatas.length - 1] || { id: -1 };
              addValue.id = lastData.id + 1;
              allDatas.push(addValue);
              return allDatas;
          }, force);
      }
  
      /**
       * * remove data by data id
       * @param {number} id 
       * @param {Boolean} force force do this function
       */
      remove(id, force) {
          id = parseInt(id, 10);
          return this._edit(allDatas => {
              return allDatas.filter(d => d.id !== id);
          }, force);
      }
  
      /**
       * * update data which has same id
       * @param {Object} newData 
       * @param {Boolean} force force do this function
       */
      update(data, force) {
          data.id = parseInt(data.id, 10);
          return this._edit(allDatas => {
              for (let i = 0; i < allDatas.length; i++) {
                  if (allDatas[i].id === data.id) {
                      for (const key in data) {
                          allDatas[i][key] = data[key];
                      }
                      break;
                  }
              }
              return allDatas;
          }, force);
      }
  
      /**
       * * overwrite the data(which in stack) and save to json db file
       * @param {Function} success 
       * @param {Function} fail
       * @param {Boolean} needSave true: save data to file, false: just update cache data
       * @param {Boolean} force force do this function, it may clear all data
       */
      save(success, fail, skipSaveFile, force) {
          if (!force && this._state === 'init') {
              throw new Error('There no data to save, or set force ro clear all data')
          }
  
          this._state = 'send';
  
          this._stack.then(allDatas => {
              this._cacheDB.data = allDatas;
              if (skipSaveFile) {
                  return allDatas;
              } else {
                  return this._writeContent(JSON.stringify(allDatas));
              }
          }).then(data => {
              success && success(data);
              return data;
          }, fail);
  
          return this;
      }
  }
  
  module.exports = JsonDatabase;
  ```

<br>



## 结尾

就目前来说，实现了我基本想要的功能，但实际使用后还是遇到和预想到了不少问题。

1. 每次全部读写整个文件，注定单文件不能存放太多的数据。
2. 需要对特殊字符进行下转义
3. 数据过滤在一些> < >= 的情况下反而麻烦了不少
4. ...

不过本来就是设计用来写写demo的，也算够了

​	