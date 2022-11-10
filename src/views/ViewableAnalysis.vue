<template>
  <div id="map"></div>
</template>

<script lang="ts" setup>
import * as maptalks from "maptalks";
import {
  GroupGLLayer,
  Geo3DTilesLayer,
  ViewshedAnalysis,
  pointAtResToAltitude,
  pointAtResToCoordinate,
  getGLRes,
} from "@maptalks/gl-layers";
import { onMounted } from "vue";
import * as dat from "dat.gui";

type IMapConfig = {
  center: number[];
  centerCross: boolean;
  doubleClickZoom: boolean;
  baseLayer: any;
  zoom: number;
  lights: any;
};

type ICoordinate = {
  x: number;
  y: number;
  z: number;
};

let groupLayer: GroupGLLayer;
let map: maptalks.Map;
let viewshedAnalysis: ViewshedAnalysis = null,
  eyePos,
  lookPoint,
  verticalAngle,
  horizontalAngle;

const mapConfig: IMapConfig = {
  center: [118.8788535, 28.9115894],
  centerCross: false,
  doubleClickZoom: false,
  baseLayer: new maptalks.TileLayer("tile", {
    urlTemplate: "https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png",
    subdomains: ["a", "b", "c", "d"],
    attribution:
      '&copy; <a href="http://osm.org">OpenStreetMap</a> contributors, &copy; <a href="https://carto.com/">CARTO</a>',
  }),
  zoom: 15,
  lights: {
    directional: {
      color: [1, 1, 1],
      lightColorIntensity: 5000,
      direction: [1, -0.4, -1],
    },
  },
};

const initMap = () => {
  map = new maptalks.Map("map", mapConfig);

  const sceneConfig = {
    environment: {
      enable: true,
      mode: 1,
      level: 1,
      brightness: 1,
    },
    shadow: {
      enable: true,
      opacity: 0.5,
      color: [0, 0, 0],
    },
    postProcess: {
      enable: true,
      antialias: {
        enable: true,
      },
      ssr: {
        enable: true,
      },
      bloom: {
        enable: true,
      },
      outline: {
        enable: true,
      },
    },
    ground: {
      enable: true,
      renderPlugin: {
        type: "lit",
      },
      symbol: {
        polygonOpacity: 1,
        material: {
          baseColorFactor: [0.48235, 0.48235, 0.48235, 1],
          hsv: [0, 0, -0.532],
          roughnessFactor: 0.22,
          metallicFactor: 0.58,
        },
      },
    },
  };
  groupLayer = new GroupGLLayer("g", [], {
    sceneConfig,
  }).addTo(map);

  loadTileset();

  analysisIntersect();
};

const loadTileset = () => {
  const thrDlayer = new Geo3DTilesLayer("3d-tiles", {
    services: [
      {
        url: "http://localhost:9003/model/Qd8heKRv/tileset.json",
        ambientLight: [1, 1, 1],
        maximumScreenSpaceError: 1.0,
        pointOpacity: 0.5,
        pointSize: 3,
        heightOffset: -30,
      },
    ],
  }).addTo(groupLayer);

  thrDlayer.once("loadtileset", (e: any) => {
    const extent = thrDlayer.getExtent(e.index);
    map.fitExtent(extent, 0);
    const center = map.getCenter();
    // 相机位置
    eyePos = [center.x + 0.003, center.y, 50];
    // 目标点
    lookPoint = [center.x, center.y, 20];
    // 垂直视角
    verticalAngle = 30;
    // 水平视角
    horizontalAngle = 20;
    // 创建可视域分析对象
    viewshedAnalysis = new ViewshedAnalysis({
      eyePos,
      lookPoint,
      verticalAngle,
      horizontalAngle,
    });
    viewshedAnalysis.addTo(groupLayer);
  });
};

const analysisIntersect = () => {
  // 以下是绘制可视域分析的交互逻辑
  let altitudes: number[] = [],
    coordinates: any[] = [],
    first = true;
  const drawTool = new maptalks.DrawTool({
    mode: "LineString",
    symbol: {
      lineOpacity: 0,
    },
  })
    .addTo(map)
    .disable();

  drawTool.on("mousemove", (e) => {
    const coordinate: ICoordinate = getPickedCoordinate(e.coordinate);
    if (!coordinate) {
      return;
    }
    if (first) {
      coordinates.push([coordinate.x, coordinate.y]);
      altitudes.push(coordinate.z);
    } else {
      coordinates[coordinates.length - 1] = [coordinate.x, coordinate.y];
      altitudes[altitudes.length - 1] = coordinate.z;
    }
    e.geometry.setProperties({
      altitude: altitudes,
    });
    e.geometry.setCoordinates(coordinates);
    lookPoint = [coordinate.x, coordinate.y, coordinate.z];
    viewshedAnalysis.update("lookPoint", lookPoint);
    first = false;
  });

  drawTool.on("drawvertex", (e) => {
    const coordinate = getPickedCoordinate(e.coordinate);
    if (!coordinate) {
      return;
    }
    if (first) {
      coordinates.push([coordinate.x, coordinate.y]);
      altitudes.push(coordinate.z);
      first = false;
    } else {
      coordinates[coordinates.length - 1] = [coordinate.x, coordinate.y];
      altitudes[altitudes.length - 1] = coordinate.z;
      first = true;
    }
    e.geometry.setProperties({
      altitude: altitudes,
    });
    e.geometry.setCoordinates(coordinates);
    lookPoint = [coordinate.x, coordinate.y, coordinate.z];
    viewshedAnalysis.update("lookPoint", lookPoint);
    drawController.setValue(false); // disable drawtool
  });

  drawTool.on("drawstart", (e) => {
    const coordinate = getPickedCoordinate(e.coordinate);
    if (!coordinate) {
      return;
    }
    coordinates.push([coordinate.x, coordinate.y]);
    altitudes.push(coordinate.z);
    e.geometry.setProperties({
      altitude: altitudes,
    });
    e.geometry.setCoordinates(coordinates);
    eyePos = [coordinate.x, coordinate.y, coordinate.z];
    viewshedAnalysis.update("eyePos", eyePos);
    first = true;
  });

  function getPickedCoordinate(coordinate: any) {
    const identifyData = groupLayer.identify(coordinate)[0];
    const pickedPoint = identifyData && identifyData.point;
    if (pickedPoint) {
      const altitude = pointAtResToAltitude(pickedPoint[2], getGLRes());
      const coordinate = pointAtResToCoordinate(
        new maptalks.Point(pickedPoint[0], pickedPoint[1]),
        getGLRes()
      );
      return new maptalks.Coordinate(coordinate.x, coordinate.y);
    } else {
      return coordinate;
    }
  }

  // dat gui的创建代码
  const gui = new dat.GUI({
    width: 250,
  });

  class Config {
    verticalAngle: number;
    horizonalAngle: number;
    draw: boolean;
    constructor() {
      this.verticalAngle = 30;
      this.horizonalAngle = 20;
      this.draw = false;
    }
  }
  const options = new Config();
  const verticalAngleController = gui.add(options, "verticalAngle", 0, 90);
  verticalAngleController.onChange(function (value) {
    viewshedAnalysis.update("verticalAngle", value);
  });
  const horizonalAngleController = gui.add(options, "horizonalAngle", 0, 90);
  horizonalAngleController.onChange(function (value) {
    viewshedAnalysis.update("horizontalAngle", value);
  });

  const drawController = gui.add(options, "draw");
  drawController.onChange(function (value) {
    if (value) {
      drawTool.enable();
    } else {
      drawTool.disable();
    }
  });
};

onMounted(() => {
  initMap();
});
</script>

<style>
#map {
  width: 100vw;
  height: 100vh;
}
</style>
