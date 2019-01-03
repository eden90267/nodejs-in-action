# Chap 02. Egg.js 框架核心原理與實現

## 異步基礎

### 什麼是異步？

放下暫時無法完成的事情，而去做可以立即完成的事情，這就是異步

### 回調實現異步

實現異步的本質是回調

```javascript
const fs = require('fs');
let data1 = fs.readFileSync('./mock.txt');

console.log(data1.toString());

let data2 = fs.readFile('./mock.txt', (err, data3) => {
  console.log('data3 ++++');
  console.log(data3.toString());
});

console.log(data2);
```

以上這種透過回調實現異步的情況，可以說在 Node.js 的 API 中比比皆是，這種原始方式容易產生詬病，當系統變大，異步因為無法直接返回值，導致異步只能跟著另一個異步，久而久之嵌套越來越多，代碼將會無法維護，這通常被開發者稱為回調地獄。

回調錯誤不太好處理，所以默認第一個參數為錯誤，假如有第一個參數，則說明發生了錯誤，那麼就要進行相對應的處理。

```javascript
function my_async_function(name, fn) {
  setTimeout(() => {
    fn(null, '-' + name + '-')
  }, 3000)
}

my_async_function('hello node.js', (err, name) => {
  if (err) {
    console.error(err)
  }
  console.log(name)
})
```

### 事件實現異步

前端瀏覽器最常見的就是頁面交互的點擊事件，在後端 Node.js 環境中，API 也提供了 events 模塊，現在實現一個事件類，事件的本質就是發佈訂閱設計模式，之所以能異步，還是因為回調。

```javascript
class Evente {
  constructor() {
    this.map = {};
  }
  add(name, fn) {
    if (this.map[name]) {
      this.map[name].push(fn);
      return;
    }
    
    this.map[name] = [fn];
    return;
  }
  emit(name, ...args) {
    this.map[name].forEach(fn => {
      fn(...args);
    });
  }
}

let e = new Evente();
e.add('hello', (err, name) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(name);
})

e.emit('hello', '發生了錯誤')
e.emit('hello', null, 'hello nodejs')
```

事件模組是學習 Node.js 的基礎，Node.js 的不少模組都是基於事件模組構建的，因為事件的裡面還是回調，所以對於處理錯誤，還是把第一個參數作為錯誤進行傳遞。

對於事件，通常會有一個名字，比如單擊事件，所以有 name 對應參數。

上面代碼其實還可做一點優化：

```javascript
class ChainEvente {
  constructor() {
    this.map = {};
  }
  add(name, fn) {
    if (this.map[name]) {
      this.map[name].push(fn);
      return this;
    }
    this.map[name] = [fn];
    return this;
  }
  emit(name, ...args) {
    this.map[name].forEach(fn => {
      fn(...args);
    });
    return this;
  }
}

let e2 = new ChainEvente();

e2.add('hello', (err, name) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(name);
})
.emit('hello', '發生了錯誤')
.emit('hello', null, 'hello nodejs')
```

把事件加入異步編程裡面，只需要在異步的方法裡面觸發這個事件即可。其實本質上都是回調，只是換了個形式而已，並沒有從根本上解決這個問題，對於簡單的異步用事件甚至感覺變麻煩了，代碼如下：

```javascript
const fs = require('fs');

function readFn(err, data) {
  console.log(data.toString());
}
fs.readFile('mock.txt', readFn);

let e2 = new ChainEvente();
e2.add('readFn', readFn);
fs.readFile('mock.txt', (err, data) => {
  e2.emit('readFn', err, data);
})
```

### 觀察者模式實現異步

```javascript
function create(fn) {
  let ret = false;
  return ({next, complete, error}) => {
    function nextFn(...args) {
      if (ret) {
        return;
      }
      next(...args);
    }
    function completeFn(...args) {
      complete(...args);
      ret = true
    }
    function errorFn(...args) {
      error(...args);
    }
    
    fn({
      next: nextFn,
      complete: completeFn,
      error: errorFn
    });
    
    return () => (ret = true)
  };
}

let observerable = create(observer => {
  setTimeout(() => {
    observer.next(1);
  }, 1000);
  observer.next(2);
  observer.complete(3);
});

const subject = {
  next: value => {
    console.log(value)
  },
  complete: console.log,
  error: console.log
};

let unsubscribe = observerable(subject)

// result:
// 2
// 3
```

上述 API 借鑒于 Rx.js，Rx.js 相當於 Promise 的增強版。

### Promise 異步

Promise 是 ES6 的新特性，專門為了處理異步而生，本質上還是回調，只不過 Promise 成為 JavaScript 的標準，所以我們通常會用 Promise 來實現異步編程。

在瀏覽器端，有的低版本瀏覽器沒有 Promise，那就需要添加 Promise 的兼容實現庫，比如 promise-polyfill、babel-polyfill 等。

```javascript
const getName = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('node.js');
  }, 50);
});

const getNumber = Promise.resolve(1);
const getError = Promise.reject('出錯啦~');

getError.catch(console.log)

Promise.all([getName, getName])
.then(console.log)
.catch(console.log);

Promise.race([getName, getName])
.then(console.log)
.catch(console.log);

getName
.then(name => {
  console.log(name);
  return 20;
})
.then(number => {
  console.log(number);
})

// result
// 出錯啦~
// node.js
// ['nodejs', 'nodejs']
// node.js
// 20
```

Promise.all 最常用，all 接受一個 Promise 物件的陣列，當所有 Promise 都返回的時候才會執行 then 後面的方法，有一個 Promise 錯誤，則都將失敗而調用 catch 裡面的回調。

Promise.race 則是只會返回一個最快完成的 Promise，也就是誰快就返回誰。

Promise 透過 new Promise 創建，需要傳入一個回調，回調有兩個參數：

- resolve：返回正確的值，將被 then 方法裡面的回調補獲
- reject：返回錯誤，會被 catch 裡面的回調捕獲

Promise 可透過靜態方法 resolve 和 reject 快速構建出一個 Promise。

Promise 鏈一旦開始，返回值會不停地用 Promise 物件重新包裏，只能不停地 “then”。

### async/await “大殺器”

Node.js 8.9 版後就支持 async/await 關鍵字，假如在前端，可用 babel 轉譯 async/await 關鍵字。異步的本質就是回調，async/await 可以在語法層面上規避回調，代碼如下：

```javascript
async function func() {
  return 2;
}

func().then(console.log)

const getPosts = () =>
  new Promise((resolve, reject) => {
    resolve([
      {name: 'a'},
      {name: 'b'},
      {name: 'c'},
      {name: 'd'},
    ]);
  });

async function func2() {
 try {
   const number = await func();
   const posts = await getPosts();
   console.log(number);
   console.log(posts);
 } catch(e) {
   console.log(e)
 }
}

func2();

// result
// 2
// 2
// [{name: "a"}, {name: "b"}, {name: "c"}, {name: "d"}]
```

async 修飾 function，說明這是一個異步的方法，比如 func 函數，當調用的時候，它會返回一個 Promise，只有透過 then 方法才能獲取返回的 2。

對於異步 func2，使用到 await 關鍵字，當調用 func 異步函數的時候，返回 Promise，而把 await 放在 Promise 前面，就可獲取被 Promise 包裏的值。

通常 async 與 await 成對出現，await 後面可以跟 Promise 和其他 async 函數，也可以跟普通的同步函數。假如其後跟的是普通的同步函數，則行為和普通同步函數一樣。

使用 async/await 之後，就可直經透過 try/catch 來補獲錯誤。

### 小結

現在我們實現異步編程的選擇是 async/await 加上 Promise。那使用 Promise 如何兼容以前的回調？兩種方式解決：

- 自己封裝
- npm 安裝其他人封裝好的

```javascript
const fs = require('fs')

const readFilePromise = filename =>
  new Promise((resolve, reject) => {
    fs.readFile(filename, (err, data) => {
      if (err) {
        reject(err);
        return;
      }
      resolve(data);
    })
  });
  
async function main() {
  const txt = await readFilePromise('mock.txt');
  console.log(txt.toString())
}

main();
```

## Koa.js 基礎知識
