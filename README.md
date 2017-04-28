*本功能基于leaflet开发，并对其示例代码做了相应整理，便于增删查改及与后端数据库交互*



# 关于使用： #

        /*打开相应弹出框*/
    function showDesc() {
        var id=saveData[0].id;
        map._layers[id].openPopup();
    }

    /*版本信息*/
    var versions=[
        {id:"001",name:"第一版",zoomLevel:6,path:"http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"},
        {id:"002",name:"第二版",zoomLevel:4,path:"http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png'"}
    ];

    /*待存储数据*/
    var saveData=[];

    /*地图参数设置*/
    var mapOption={
        center: new L.LatLng(0, 0), //地图的初始地理中心
        zoomControl: false, //放大缩小按钮是否显示
        zoom:1, //初始地图缩放级别
        minZoom:0, //地图的最小缩放级别
        scrollwheel: true,
        legends: true,
        infoControl: false,
        attributionControl: false, //是否显示版权信息
        preferCanvas:false  //是否Path应在渲染Canvas器上渲染。默认情况下，所有Paths都在SVG渲染器中呈现
    };

    /*版本参数设置*/
    var tileLayerOption={
        tileSize:256,
        noWrap:true //是否环绕
    };

    /*创建地图*/
    var map = new L.Map('map', mapOption);

    /*批注信息*/
    var drawnItems = L.featureGroup().addTo(map);

    /*初始化图层信息*/
    var layers={};
    versions.map(function(item,index){
        layers[item.name]=L.tileLayer(item.path,  Object.assign({}, tileLayerOption, {maxZoom:item.zoomLevel,id:item.id}));
        if(index==0){
            layers[item.name].addTo(map);
        }
    });
    L.control.layers(layers, { '显示批注信息': drawnItems }, { position: 'topright', collapsed: true }).addTo(map);

    /*放大缩小按钮*/
    L.control.zoom({ position: "bottomleft"}).addTo(map);

    /*工具初始化*/
    var drawTool=new L.Control.Draw({
        edit: {
            featureGroup: drawnItems,
            poly: {
                allowIntersection: false
            }
        },
        draw: {
            polyline: {
                shapeOptions: {
                    color: "red",
                    //opacity: 0.5,
                    //weight: 4
                }
            },
            polygon: {
                shapeOptions: {
                    color: "red",
                    //weight: 4,
                    //opacity: 0.5,
                },
                allowIntersection: false,
                showArea: true
            },
            rectangle:{
                shapeOptions: {
                    color: "red",
                    //weight: 4,
                    //opacity: 0.5,
                }
            },
            circle:{
                shapeOptions: {
                    color: "red",
                    //weight: 4,
                    //opacity: 0.5
                }
            },
            marker: {
                icon: new L.Icon.Default()
            }
        }
    });
    map.addControl(drawTool);

    /*全屏*/
    map.addControl(new L.Control.Fullscreen());

    var popupState;
    /*添加图层*/
    map.on(L.Draw.Event.CREATED, function (event) {
        var layer = event.layer;
        drawnItems.addLayer(layer);

        /*注册绑定事件*/
        layerClick(layer);

        /*保存数据*/
        saveData.push({
            id: layer._leaflet_id,
            layerType:event.layerType,
            style:layer.options,
            geoJSON: layer.toGeoJSON()
        });

        var popup = layer.bindPopup("<div>添加图层</div>");
        popup.openPopup();

        /*阻止弹出框关闭*/
        /*        popupState=true;
         popup.on("popupclose", function(e) {
         var that= this;
         if(popupState){
         setTimeout(function () {
         that.openPopup();
         },1);
         }
         });*/

        console.log("添加图层",layer,event.layerType);
        window.localStorage.setItem("savedItems",JSON.stringify(saveData))
    });

    /*编辑图层*/
    map.on(L.Draw.Event.EDITED,function (event) {
        var layers = event.layers;
        layers.eachLayer(function(layer) {
            saveData.map(function(item,index){
                if (item.id == layer._leaflet_id) {
                    item.geoJSON = layer.toGeoJSON();
                }
            })
        });
        console.log("编辑图层",layers);
        window.localStorage.setItem("savedItems",JSON.stringify(saveData))
    });

    /*删除图层*/
    map.on(L.Draw.Event.DELETED , function (event) {
        var layers = event.layers;
        layers.eachLayer(function(layer) {
            saveData.map(function(item,index){
                if (item.id == layer._leaflet_id) {
                    saveData.splice(index,1);
                }
            })
        });
        console.log("删除图层",layers);
        window.localStorage.setItem("savedItems",JSON.stringify(saveData))
    });

    /*点击图层*/
    function layerClick(layer) {
        layer.on({
            click: function (e) {
                var html='<div class="layer-box">' + this._leaflet_id + '</div>';

                /*点击弹出详情框*/
                this.bindPopup(html);

                /*解决弹框不响应bug*/
                if(!this._popup._container){
                    var popup= L.popup({}).setContent(this._popup._content);
                    this.bindPopup(popup).openPopup();
                }
            }
        });
    }

    /*初始化数据*/
    var init=function() {
        saveData=JSON.parse(window.localStorage.getItem("savedItems"))||[];
        saveData.map(function(item,index){
            L.geoJson(item.geoJSON, {
                pointToLayer: function (feature, latlng) {
                    if(item.layerType==L.Draw.Circle.TYPE){
                        return L.circle(latlng, item.style);
                    }else if(item.layerType==L.Draw.Marker.TYPE){
                        return L.marker(latlng, {icon: new L.Icon.Default()});
                    }
                },
                style: function(feature) {
                    return item.style;
                },
                onEachFeature: function (feature, layer) {
                    if (layer.getLayers) {
                        layer.getLayers().forEach(function (l) {
                            l._leaflet_id=item.id;
                            drawnItems.addLayer(l);

                            /*注册绑定事件*/
                            layerClick(l);
                        })
                    } else {
                        layer._leaflet_id=item.id;

                        /*注册绑定事件*/
                        layerClick(layer);
                        //layer.editing.enabled();
                        drawnItems.addLayer(layer);
                    }
                }
            });
        })
    }();

# 版权申明： #

本功能基于leaflet二次开发，仅为个人学习使用，如商用请遵循leaflet相关版权规则。

# 参考： #

1. http://leaflet.github.io/Leaflet.draw/docs/leaflet-draw-latest.html
2. http://blog.mastermaps.com/2013/06/showing-zoomify-images-with-leaflet.html
3. https://github.com/asfez/CsZoomify
4. http://openlayers.org/