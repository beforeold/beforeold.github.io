在地图服务的应用场景中，有一个常见的需求是在地图中心放置一个大头针，让用户拖动地图，从地图上选择一个地点，要求：这个大头针在地图的拖动过程中不移动位置。

常规的情况下，大头针视图作为地图的一部分，是会随着地图移动而移动，那么解决的思路可以反向推导为两种方式：
- 中心大头针与地图视图分离
- 在地图移动的过程中不断修改大头针视图的坐标，使其一直处于地图中心

思路确定后，实现即相对容易了：
- 第一种：将一个imageView添加在地图或者地图的父视图上，在地图拖动结束后获取当前地图中心点的位置信息；
第二种：为中心大头针设置一个单独的annotation，在地图拖动的回调中不断修改这个标注annotation的坐标coordinate即可，从接触的苹果、高德和百度的地图SDK看来，目前只有百度SDK有针对地图拖动的即时性回调方法：

```objective-C
- (void)mapView:(BMKMapView *)mapView onDrawMapFrame:(BMKMapStatus *)status {
      self.annotatioin.coordinate = mapView.centerCoordinate;
}
```
对于苹果地图、高德地图则可以依赖地图区域变化的回调来实现，地图变动后将坐标进行设置，无法做到像百度那样的连续。
```objctive-C
- (void)mapView:(MAMapView *)mapView regionDidChangeAnimated:(BOOL)animated {
      self.annotatioin.coordinate = mapView.centerCoordinate;
}
```

注意有一些方案提出监听地图status变化（百度）的回调，这样会导致过度的监听，没有必要。

- 这个情况就别问demo啦。
