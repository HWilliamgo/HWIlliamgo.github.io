1. 当RecyclerView组装完后，立刻去获取Item的高度：
```
int childHeight = recyclerView.getLayoutManager().findViewByPosition(0).getHeight();
```
会报空指针，因为itemView还没有attach到recyclerView中，获取不到itemView，用`view.post()`解决
``` java
recyclerView.post(new Runnable){
    int childHeight = recyclerView.getLayoutManager().findViewByPosition(0).getHeight();
}
```
