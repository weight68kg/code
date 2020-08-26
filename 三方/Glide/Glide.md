
```sequence

Glide-> Glide:getRetriever

Glide-> GlideBuild : 
activate GlideBuild
Note left of Glide: 调用with
Note over Glide: 
GlideBuilder -> RequestManagerRetriever:
RequestManagerRetriever -> RequestManagetFactory :构造函数中初始化factory
RequestManagetFactory -> RequestManager : build()


```

```mermaid
sequenceDiagram
对象A->对象B:中午吃什么？
对象B->>对象A: 随便
loop 思考
    对象A->对象A: 努力搜索
end
对象A-->>对象B: 火锅？
对象B->>对象A: 可以
Note left of 对象A: 我是一个对象A
Note right of 对象B: 我是一个对象B
participant 对象C
Note over 对象C: 我自己说了算


```




Glide.with(Context)最后返回 RequestManager,其中经过创建 RequestManagerRetriever,

