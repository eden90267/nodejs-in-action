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
