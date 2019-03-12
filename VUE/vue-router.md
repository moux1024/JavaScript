# Something about vue-router

## vue-router的replace
> 今天遇到的情况是:

````js
this.$router.replace({path:"xxx",Object.assign(this.$route.query, { activity: "share6" }))
````

操作不更新href
其实原因很简单,assign修改了target,router在replace时compare发现,新旧route是完全一样的,当然就不会触发更新,这个操作是标准的:越过v操作m,mv层当然一点动静都没有
