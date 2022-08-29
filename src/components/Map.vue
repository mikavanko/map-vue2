<script>
import { reactive, onMounted, ref, nextTick } from "vue";
import Pie from "@/components/Pie.vue";

import moscowJson from "../assets/moscow.json";
import moscowDistrictsJson from "../assets/moscow-districts.json";

import randomPositionInPolygon from "random-position-in-polygon";

const moscowCoords = [37.6156, 55.7522];
const apiKey = "92d57784-7d4e-48b9-b796-89b8ff88f1d1";

export default {
  components: {
    Pie,
  },
  emits: ["regenerate-coords"],
  props: {
    geoObjectsCount: {
      type: Number,
      default: 100,
    },
  },
  setup(props, { emit }) {
    const state = reactive({
      mapCenter: [...moscowCoords],
      map: null,
      ymaps: null,
      geoObjectCollection: null,
      districtZones: null,
      coordsByRegions: new Array(moscowDistrictsJson.features.length),
      coordsByRegionsCount: {},
      activeRegions: [],
    });

    onMounted(() => {
      loadYandexMap(() => {
        ymaps.ready(() => {
          state.map = new ymaps.Map(
            "map",
            {
              center: state.mapCenter,
              zoom: 10,
              controls: ["zoomControl"],
            },
            {
              searchControlProvider: "yandex#search",
            }
          );
          state.districtZones = addJsonOnMap(state.map, moscowDistrictsJson);
          state.geoObjectCollection = generateRandomGeoObjects(
            props.geoObjectsCount,
            state.map
          );
        });
      });
    });

    function loadYandexMap(callback) {
      let yandexMapScript = document.createElement("script");
      yandexMapScript.setAttribute(
        "src",
        `https://api-maps.yandex.ru/2.1/?lang=ru_RU&coordorder=longlat&apikey=${apiKey}`
      );
      document.head.appendChild(yandexMapScript);

      yandexMapScript.addEventListener("load", () => {
        callback();
      });
    }

    function addJsonOnMap(map, json) {
      const districtCollection = new ymaps.GeoObjectCollection(
        {},
        {
          preset: "islands#redCircleIcon",
          strokeWidth: 4,
          geodesic: true,
        }
      );

      json.features.forEach((f) => {
        const districtGeoObject = new ymaps.GeoObject(f, {
          id: f.id,
          fillColor: f.properties.fill,
          strokeColor: f.properties.stroke,
          opacity: f.properties["fill-opacity"],
          strokeWidth: f.properties["stroke-width"],
        });

        districtCollection.add(districtGeoObject);
      });

      return ymaps.geoQuery(districtCollection).addToMap(map);
    }

    function generateRandomGeoObjects(count, map) {
      const moscowPolygon = moscowJson.features[0];

      const myGeoObjects = new ymaps.GeoObjectCollection(
        {},
        {
          preset: "islands#redCircleIcon",
          strokeWidth: 4,
          geodesic: true,
        }
      );

      for (let i = 0; i < count; i++) {
        const placemark = new ymaps.Placemark(
          randomPositionInPolygon(moscowPolygon)
        );
        const placemarkCoords = placemark.geometry.getCoordinates();
        const polygon = state.districtZones
          .searchContaining(placemarkCoords)
          .get(0);

        if (polygon) {
          if (state.coordsByRegions[polygon.options.get("id")]) {
            state.coordsByRegions[polygon.options.get("id")].push(placemark);
          } else {
            state.coordsByRegions[polygon.options.get("id")] = [placemark];
          }
          myGeoObjects.add(placemark);
        }
      }

      const coordsByRegionsCount = {};

      for (const key in state.coordsByRegions) {
        coordsByRegionsCount[key] = state.coordsByRegions[key].length;
      }

      state.coordsByRegionsCount = coordsByRegionsCount;

      map.geoObjects.add(myGeoObjects);

      return myGeoObjects;
    }

    function regenerateCoords() {
      state.coordsByRegions = new Array(moscowDistrictsJson.features.length);
      state.coordsByRegionsCount = {};
      state.activeRegions = [];
      state.geoObjectCollection.removeAll();
      state.geoObjectCollection = generateRandomGeoObjects(
        props.geoObjectsCount,
        state.map
      );

      emit("regenerate-coords");
    }

    function toggleRegion(region) {
      if (state.activeRegions.includes(region.id)) {
        state.coordsByRegions[region.id].forEach((p) => {
          p.options.set("visible", true);
        });
        const idx = state.activeRegions.indexOf(region.id);
        if (idx !== -1) {
          state.activeRegions.splice(idx, 1);
        }
      } else {
        state.coordsByRegions[region.id].forEach((p) => {
          p.options.set("visible", false);
        });
        state.activeRegions.push(region.id);
      }
    }

    return {
      state,
      regenerateCoords,
      moscowDistrictsJson,
      toggleRegion,
    };
  },
};
</script>
      
<template>
  <div class="map-wrapper flex">
    <div class="flex-child">
      <div id="map" class="map" />
    </div>
    <div class="pie-wrapper flex-child">
      <Pie
        class="pie-chart"
        :features="moscowDistrictsJson.features"
        :regions-count="state.coordsByRegionsCount"
        @click-section="toggleRegion"
      />
      <button class="generate-random-btn" @click="regenerateCoords">
        Перегенерировать координаты
      </button>
    </div>
  </div>
</template>
    
<style lang="scss">
.map {
  height: 100%;
  width: 100%;
}
.flex {
  display: flex;
  justify-content: flex-start;
  align-items: flex-start;
  height: 100vh;
}
.flex-child {
  width: 50%;
  height: 100%;
  min-height: 100%;
}
.pie-wrapper {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  flex-direction: column;
  gap: 50px;
}
.pie-chart {
  padding: 30px;
  flex: 1;
  width: 100%;
  height: 100%;
}
.generate-random-btn {
  @include submit-button;
  background-color: $color-dark-70;
  border-radius: 0;
  height: 15vh;
  min-height: 100px;
  transition: 0.3s ease;
  font-size: 30px;
  line-height: 1.5;
  padding: 20px;
  box-shadow: 0px 0px 20px rgba(34, 34, 34, 0.5);
  z-index: 7;

  &:hover {
    opacity: 0.9;
  }
  &:active {
    opacity: 1;
  }
}

@media all and (max-width: 768px) {
  .flex {
    height: 100vh;
    flex-direction: column;
    overflow: hidden;
  }
  .flex-child {
    width: 100%;
    height: 50%;
    min-height: 0;
  }
  .map {
    height: 100%;
    width: 100%;
  }
  .generate-random-btn {
    font-size: 16px;
    height: 60px;
    height: 20%;
    min-height: 0;
  }
  .pie-chart {
    padding: 10px;
    flex: 0;
    height: 80%;
  }
  .pie-wrapper {
    gap: 10px;
  }
}
</style>