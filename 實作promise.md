# promise
實作promise
```javascript
function Promise(fn) {

  let status = 'PENDING';
  // 存儲當前promise resolve的值
  let value;
  // callback暫存器
  let latency = null;

  function resolve(newValue) {
    // 判斷上一個傳遞下來的值(value)是否為promise
    // 若為promise，則resolve該值
    if (typeof newValue === 'object' && typeof newValue.then === 'function') {
      newValue.then(resolve, reject);
      return;
    }
    value = newValue;
    status = 'RESOLVED';

    if(latency) handle(latency);
  }

  function reject(reason) {
    value = reason;
    status = 'REJECTED';

    if(latency) handle(latency);
  }

  function handle(handler) {
    let cb;
    
    if(status === 'PENDING') {
      latency = handler;
      return
    }
    
    if(status === 'RESOLVED') {
      cb = handler.ResolveCallback;
    } else {
      cb = handler.RejectCallback;
    }

    setTimeout(() => {
      if (!cb) {
        // status已經更動完成，因此resolve/reject並無多大區別(resolve多了解析promise的作用)
        handler.resolve(value);
        return;
      }

      let ret = cb(value);
      // 使用resolve更新value，status在外面的promise物件呼叫方法(resolve/reject)時已經更動完成
      handler.resolve(ret);
    }, 0)
  }

  this.then = function(ResolveCallback, RejectCallback) {
    return new Promise(function(resolve) {
      // handle為連繫then(兩個promise)間的橋梁，帶入的是前一個promise的handle及其closure
      handle({ reject, resolve, ResolveCallback, RejectCallback });
    });
  }
  
  fn(resolve, reject);
}
```  

## 結構及功用：
* resolve/reject: 更新下個promise可使用的value及status(並解析promise物件)
* handle: 用來呼叫、條件式的緩存callback
* then: 連繫下一個then，並運用前一個promise的function及變數們(handle的closure)

  > 為了能連續的串聯串聯then，直接return一個新建立的promise物件  
  ex: ```a.then().then() => b.then(), b == a.then()```b為a.then()方法所return出一個不可見的promise物件
    
* 初始化: ```... fn(resolve, reject) ...```:：將定義好的resolve及reject傳到外面的promise物件  

## 舉例
```javascript
let doSomething = new Promise(function(a,b) {
  a('aaaaaa')
});
let partOfSomething = doSomething.then(console.log)

partOfSomethin.then(console.log).then(v => 'bbbbbb').then(console.log)

/* 輸出結果: 
 *
 * aaaaaa
 * undefined
 * bbbbbb 
 *
 */
```  

## 流程:
1. ```fn(resolve, reject)```

   將類別promise內部定義的resolve/reject傳遞出去，供外界new的promise物件:doSomething使用
   
1. doSomething呼叫a，也就是resolve方法
1. ```resolve('aaaaaa')```

   將value, status更新為'aaaaaa', 'RESOLVED'
   
1. ```this.then(...)```

   解析then，回傳一個promise物件傳給partOfSomething，並呼叫 **當前的(doSomething)** 這個promise的handle
   > 為的是讓partOfSomething也能存取到前一個promise已經解析完的值(value)
   
1. ```handle({ ... })```

   執行handle
1. ```if(status === 'PENDING') {...}```

   若status仍為PENDING，代表尚未resolve/reject，因此先將callback暫存在latency變數
   等resolve載入時，再呼叫latency(callback)，並將value帶入
   
1. setTimeout已確保非同步
1. ```if (status === 'RESOLVED') { ... } else { ... }```

   根據doSomething的狀態(a('aaaaaa')更動狀態)來指定要呼叫resolve/reject callback
   
1. ```if (!cb) { ... }```

   當傳來的handler(then裡)沒有callback，則直接將value及狀態更新，供下個then的callback使用(**穿透性**)
   
1. ```
    let rt = cb(value)
    handler(rt)
   ```

   若status不是PENDING且handler有callback，則呼叫callback，將callback執行後返回的值拿去更新value
   > 若callback中沒有return一個值，則下一個promise只會收到value初始化的值: undefined
   
1. 接著解析下一個then...  

## 問題釋例
+ 為什麼要有latency?

   **不然then比resolve先做完時，不會正確的輸出**  
   jsfiddle: https://jsfiddle.net/0qwo68c2/

```javascript
...
if(latency) latency(value);
...
if(status === 'PENDING') {
  latency = handler;
  return
}
...
```
+ 為什麼要有setTimeout?

   **不然無法保證非同步**  
   jsfiddle: https://jsfiddle.net/w0j6rnyq/
    
```javascript
setTimeout(() => {
  if (!cb) {
    if (status === 'RESOLVED') {
      handler.resolve(value);
    } else {
      handler.reject(value)
    }
    return;
  }
  let ret = cb(value);
  handler.resolve(ret);
}, 0)
```

+ 為什麼要回傳promise?
   
   **不然無法一直串連then**  
   jsfiddle: https://jsfiddle.net/rkw7kx9f/

```javascript
this.then = function(callback) {
  return new Promise(function(resolve){
    // handle為連繫then(兩個promise)間的橋梁，帶入的是前一個promise的handle及其closure
    handle({ reject, resolve, callback });
  });
}
```

+ 為什麼要在resolve裡檢查是否值為promise並解析?

   **不然只會得到一個promise object，還要在then裡面再處理一次promise**  
   jsfiddle: https://jsfiddle.net/73m0bdou/
   
```javascript
function resolve(newValue) {
  if (typeof newValue === 'object' && typeof newValue.then === 'function') {
    newValue.then(resolve, reject);
    return;
  }
  ...
}
```

+ 為什麼已經存在( ..., reject){ ... }，還要有.catch(e){ ... }?

   **因為在then裡的程式有可能發生錯誤**  
   jsfiddle: https://jsfiddle.net/xc9vmqq7/

```javascript
// No alert...
new Promise(function(resolve,reject) {
  if(...) {
    resolve(...)
  } else {
    reject(...)
  }
}).then(
  function(val) {
    throw 'some unexpected errors';
  },
  function(err) {
    alert(err)
  }
)

// alert 'in catch: some unexpected errors'
new Promise(function(resolve,reject) {
  if(...) {
    resolve(...)
  } else {
    reject(...)
  }
}).then(
  function(val) {
    throw 'some unexpected errors';
  },
  function(err) {
    alert('in reject:'+err);
  }
).catch(function(e){
    alert('in catch: '+e);
})
```
  

## Reference
1. https://www.promisejs.org/implementing/
2. http://www.mattgreer.org/articles/promises-in-wicked-detail/
