<template>
  <div id="cesiumContainer">


    
  </div>
</template>
<script setup>
import * as Cesium from 'cesium';
import { onMounted } from 'vue';
onMounted(()=>{
  // const viewer = new Cesium.Viewer('cesiumContainer');
  var custom = new Cesium.ArcGisMapServerImageryProvider({
    url:'https://sampleserver1.arcgisonline.com/ArcGIS/rest/services/' +
          'Specialty/ESRI_StateCityHighway_USA/MapServer'
  })

  var viewer = new Cesium.Viewer('cesiumContainer',{
    // 不创建baseLayerPicker小部件
    baseLayerPicker:false,
    // 设置图像提供的程序
    imageryProvider:custom,
    // 设置cesium世界地形
    terrainProvider:Cesium.createWorldTerrain({
      requestWaterMask:true,
      requestVertexNormals:true,
    })
  })

  viewer.camera.setView({
    destination:new Cesium.Cartesian3(1332761,-4662399,4137888),
    orientation:{
      heading:0.60,
      pitch:-0.66
    }

    
  })

  var city = viewer.scene.primitives.add(new Cesium.Cesium3DTileset({url:Cesium.IonResource.fromAssetId(3839)}));
  
  // 定义3d样式
  var heightStyle = new Cesium.Cesium3DTileStyle({
    color:{
      // 条件判断具体的颜色
      conditions:[
        ['${height} >= 300','rgba(45,0,75,0.5)'],
        ['${height} >= 200','rgb(102,71,151)'],
        ['${height} >= 100','rgba(170,162,204,0.5)'],
        ['${height} >= 50','rgba(224,226,238,0.5)'],
        ['${height} >= 25','rgba(252,230,200,0.5)'],
        ['${height} >= 10','rgba(248,176,87,0.5)'],
        ['${height} >= 5','rgba(198,106,11,0.5)'],
        ["true","rgb(127,59,8)"]
      ]
    }
  })

  city.style = heightStyle;
  var geojsonOptions = {
        clampToGround : true
    };
    // 从 GeoJson 文件加载邻域边界
    var neighborhoodsPromise = Cesium.GeoJsonDataSource.load('./assets/SampleData/sampleNeighborhoods.geojson', geojsonOptions);
    // 保存邻域数据的新实体集合
    var neighborhoods;
    neighborhoodsPromise.then(function(dataSource) {
        // 将新数据作为实体添加到查看器
        viewer.dataSources.add(dataSource);
        neighborhoods = dataSource.entities;

        // Get the array of entities
        var neighborhoodEntities = dataSource.entities.values;
        for (var i = 0; i < neighborhoodEntities.length; i++) {
            var entity = neighborhoodEntities[i];

            if (Cesium.defined(entity.polygon)) {
                // Use kml neighborhood value as entity name
                entity.name = entity.properties.neighborhood;
                // Set the polygon material to a random, translucent color
                entity.polygon.material = Cesium.Color.fromRandom({
                    red : 0.1,
                    maximumGreen : 0.5,
                    minimumBlue : 0.5,
                    alpha : 0.6
                });
                // 告诉多边形为地形着色。 ClassificationType.CESIUM_3D_TILE 将为 3D 图块集着色，而 ClassificationType.BOTH 将为 3d 图块和地形着色（BOTH 是默认值）
                entity.polygon.classificationType = Cesium.ClassificationType.TERRAIN;
                // 生成多边形中心
                var polyPositions = entity.polygon.hierarchy.getValue(Cesium.JulianDate.now()).positions;
                // 边界球
                var polyCenter = Cesium.BoundingSphere.fromPoints(polyPositions).center;
                // 椭球体
                polyCenter = Cesium.Ellipsoid.WGS84.scaleToGeodeticSurface(polyCenter);
                entity.position = polyCenter;
                // 生成标签
                entity.label = {
                    text : entity.name,
                    showBackground : true,
                    scale : 0.6,
                    horizontalOrigin : Cesium.HorizontalOrigin.CENTER,
                    verticalOrigin : Cesium.VerticalOrigin.BOTTOM,
                    distanceDisplayCondition : new Cesium.DistanceDisplayCondition(10.0, 8000.0),
                    disableDepthTestDistance : 100.0
                };
            }
        }
    });

  var kmlOptions = {
        camera : viewer.scene.camera,
        canvas : viewer.scene.canvas,
        // 如果我们想要将几何特征（多边形、线串和线性环）固定在地面上，则为 true。
        clampToGround : true
    };
    //加载KML文件
    var geocachePromise = Cesium.KmlDataSource.load('./assets/SampleData/sampleGeocacheLocations.kml', kmlOptions);

    // 将 geocache 广告牌实体添加到场景中并为其设置样式
    geocachePromise.then(function(dataSource) {
        // 将新数据作为实体添加到查看器
        viewer.dataSources.add(dataSource);

        // 获取实体数组
        var geocacheEntities = dataSource.entities.values;

        for (var i = 0; i < geocacheEntities.length; i++) {
            var entity = geocacheEntities[i];
            if (Cesium.defined(entity.billboard)) {
                // 调整垂直原点，使图钉位于地形上
                entity.billboard.verticalOrigin = Cesium.VerticalOrigin.BOTTOM;
                entity.billboard.image = '/assets/tagpark.png'
                // 禁用标签以减少混乱
                entity.label = undefined;
                // 添加距离显示条件
                entity.billboard.distanceDisplayCondition = new Cesium.DistanceDisplayCondition(10.0, 20000.0);
                // 以度为单位计算纬度和经度
                var cartographicPosition = Cesium.Cartographic.fromCartesian(entity.position.getValue(Cesium.JulianDate.now()));
                var latitude = Cesium.Math.toDegrees(cartographicPosition.latitude);
                var longitude = Cesium.Math.toDegrees(cartographicPosition.longitude);
                // 修改描述
                var description = '<table class="cesium-infoBox-defaultTable cesium-infoBox-defaultTable-lighter"><tbody>' +
                    '<tr><th>' + "Longitude" + '</th><td>' + longitude.toFixed(5) + '</td></tr>' +
                    '<tr><th>' + "Latitude" + '</th><td>' + latitude.toFixed(5) + '</td></tr>' +
                    '<tr><th>' + "实时人流" + '</th><td>' + Math.floor(Math.random()*20000)  + '</td></tr>' +
                    '<tr><th>' + "安全等级" + '</th><td>' + Math.floor(Math.random()*5)  + '</td></tr>' +
                    '</tbody></table>';
                entity.description = description;
            }
        }
    });
  
  // 从czml文件加载飞行路径
  var dronePromise = Cesium.CzmlDataSource.load('./assets/SampleData/sampleFlight.czml');

  // 无人机实体
  var drone;
  dronePromise.then(function(dataSource){
    viewer.dataSources.add(dataSource);
    drone = dataSource.entities.getById('Aircraft/Aircraft1');
    drone.model = {
      // uri:'./assets/SampleData/Models/CesiumDrone.gltf',
      uri:'./assets/SampleData/Models/ferrari2.gltf',
      minimumPixelSize:128,
      maximumScale:1000,
      silhouetteColor:Cesium.Color.WHITE,
      silhouetteSize:2
    }

    drone.orientation = new Cesium.VelocityOrientationProperty(drone.position);
    drone.viewFrom = new Cesium.Cartesian3(0,-30,30)
    viewer.clock.shouldAnimate = true;
  })

})
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
html,body,#cesiumContainer{
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
