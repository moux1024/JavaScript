## 使用vue时发现的问题
### 情况
> 由于觉得同页面的多个模态框open&close状态管理麻烦,故想封装一个方法用于管理各个模态框status值,如下
```js
toggleStatus(obj, key) {
      // eslint-disable-next-line no-param-reassign
      obj[key] = !obj[key];
      console.log(this.status);
    }
```
调用方式为
```
@click="toggleStatus(status,'showBeastsModal')"
```
但是使用中发现,这种写法会有一定延迟,没有粗暴的
```
@click="status.showBeastsModal=false)"
```
反应快,不知道为什么
