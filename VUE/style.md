# vue内联样式的坑
> vue使用过程中我们经常需要处理内联样式,如使用function来返回一个符合我们要求的字符串来作为内联样式对应字段的值,这里通过一个莫名其妙的失败来探索一下vue的内联样式在我们不知道的地方还做了什么.
## 情景还原
```vue
<template>
  {{init_img(bg_img)}}
  <div class="mask_bg" :style="{background:`url(${init_img(bg_img)}) 100% 100%,rgba(255,255,255,0.8) center`}"></div>
</template>
<script>
export default {
  name:"test",
  data(){
    return {
      bg_img:"test(1).png"
    }
  },
  method:{
    init_img(url){
      return "http://www.baidu.com/"+bg_img
    }
  }
}
</script>
```

我们预计会得到一个url:("http://www.baidu.com/test(1).png")拼入style指令中,经过测试,init_img方法确实执行了一次,且view可以看到处理结束的字符串,然而style里只有"http://www.baidu.com/",为什么?

## 问题定位

检查了一下结果,感觉(1)非常可疑,于是使用replace删除了(1),发现style正常渲染了,虽然由于地址错误没有取到图片,但已经确认了是这里小括号的问题.
因此可以大致推定,是style在取值时,对值做了一次检查,没有通过,则直接被置为空字符串.

## 解决方案

其实这里就属于是写法不够严谨,如果在style里将属性值用单引号"'"包裹起来,就可以直接解决这个问题.因为字符串是能够通过检查的.
## 参考
- [vue的Class与Style绑定](https://cn.vuejs.org/v2/guide/class-and-style.html)
