## ListView上拉加载和下拉刷新

 



### 自定义View实现上拉加载和下拉刷新

### 使用PullToRefresh 实现上拉加载和下拉刷新

### 使用Ultra-Pull-To-Refresh实现上拉加载和下拉刷新


```java 
onUIRefreshPrepare
isUnderTouchtrueheadHeight: 100 lastPosY 0 offsetToRefresh 120 offsetY 23.025852 currentPosY 23
isUnderTouchtrueheadHeight: 100 lastPosY 299 offsetToRefresh 120 offsetY 2.1575928 currentPosY 301
onUIRefreshBegin
isUnderTouchfalseheadHeight: 100 lastPosY 301 offsetToRefresh 120 offsetY 2.1575928 currentPosY 278
isUnderTouchfalseheadHeight: 100 lastPosY 101 offsetToRefresh 120 offsetY 2.1575928 currentPosY 100
onUIRefreshComplete
isUnderTouchfalseheadHeight: 100 lastPosY 100 offsetToRefresh 120 offsetY 2.1575928 currentPosY 96
isUnderTouchtrueheadHeight: 100 lastPosY 239 offsetToRefresh 120 offsetY 1.324391 currentPosY 240
onUIRefreshComplete
isUnderTouchfalseheadHeight: 100 lastPosY 240 offsetToRefresh 120 offsetY 1.324391 currentPosY 223
isUnderTouchfalseheadHeight: 100 lastPosY 2 offsetToRefresh 120 offsetY 1.324391 currentPosY 1
onUIReset
isUnderTouchfalseheadHeight: 100 lastPosY 1 offsetToRefresh 120 offsetY 1.324391 currentPosY 0
```


### 使用SwipeToRefreshLayout实现上拉加载和下拉刷新