
    
    
```java
     /**
     * 返回true代表处理本次事件
     */
    @Override
    public boolean onStartNestedScroll(@NonNull View child, @NonNull View target, int axes, int type) {
    return false;
    }
```

```java
    /**
    * onStartNestedScroll 返回true 执行
    */
    @Override
    public void onNestedScrollAccepted(@NonNull View child, @NonNull View target, int axes, int type) {
    }
```
```java
    /**
    * 复位初始位置
    */
    @Override
    public void onStopNestedScroll(@NonNull View target, int type) {

    }
```

@Override
public void onNestedScroll(@NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int type) {

}

@Override
public void onNestedPreScroll(@NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {

}





