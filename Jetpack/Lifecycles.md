# 作用
管理您的 Activity 和 Fragment 生命周期

# 使用

```java
class MainActivity : AppCompatActivity() {
     override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        //第一种方式
        //如果不依赖 kapt "androidx.lifecycle:lifecycle-compiler:$2.2.0" 会使用反射(待我确认)
        lifecycle.addObserver(object : LifecycleObserver {

            @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
            fun onStart() {
               //onStart生命周期回调，方法名随便起，重点是注解
            }

            //省略其他5个生命周期（没有restart）
            
        })
  
        //第二种方式
        //DefaultLifecycleObserver 需要依赖 implementation 'androidx.lifecycle:lifecycle-common-java8:2.2.0'
        lifecycle.addObserver(object :DefaultLifecycleObserver{
            override fun onCreate(owner: LifecycleOwner) {
                super.onCreate(owner)
            }

            //省略其他5个生命周期 可以不重写
        })

        //第三种方式
        lifecycle.addObserver(object : LifecycleEventObserver {
            override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
                when(event){
                    Lifecycle.Event.ON_CREATE->{

                    }
                }
            }
        })
    }
}
```
