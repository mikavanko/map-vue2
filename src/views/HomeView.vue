<script>
import { reactive, onMounted, ref, nextTick } from "vue";

import moscowJson from "../assets/moscow.json";
import moscowDistrictsJson from "../assets/moscow-districts.json";

import randomPositionInPolygon from "random-position-in-polygon";

const moscowCoords = [37.6156, 55.7522];
const RANDOM_GEO_OBJECTS_COUNT = 100;

export default {
  components: {},
  setup() {
    const coordsByRegions = ref(123)
    const state = reactive({
      mapCenter: [...moscowCoords],
      map: null,
      ymaps: null,
      geoObjectCollection: null,
      districtsGeoObject: [],
      districtZones: null,
      objectManager: null,
      coordsByRegions: new Array(moscowDistrictsJson.features.length),
      coordsByRegionsCount: {},
      activeRegions: [],
    });

    // setTimeout(()=>{
    //   coordsByRegions.value = [123,123,123]
    // },1000)

    onMounted(() => {
      console.log("moscowDistrictsJson", moscowDistrictsJson);

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
          state.geoObjectCollection = generateRandomGeoObjects(RANDOM_GEO_OBJECTS_COUNT, state.map);
        });
      });
    });

    function loadYandexMap(callback) {
      let yandexMapScript = document.createElement("script");
      yandexMapScript.setAttribute(
        "src",
        "https://api-maps.yandex.ru/2.1/?lang=ru_RU&coordorder=longlat&apikey=92d57784-7d4e-48b9-b796-89b8ff88f1d1"
      );
      document.head.appendChild(yandexMapScript);

      yandexMapScript.addEventListener("load", () => {
        callback();
      });
    }

    function addJsonOnMap(map, json) {
      const districtCollection = new ymaps.GeoObjectCollection({}, {
        preset: "islands#redCircleIcon",
        strokeWidth: 4,
        geodesic: true
      });

      json.features.forEach(f => {
        console.log(f)

        const districtGeoObject = new ymaps.GeoObject(f, {
          id: f.id,
          fillColor: f.properties.fill,
          strokeColor: f.properties.stroke,
          opacity: f.properties['fill-opacity'],
          strokeWidth: f.properties['stroke-width'],
        })

        districtCollection.add(districtGeoObject)

        state.districtsGeoObject.push(districtGeoObject)

        
      })

      return ymaps.geoQuery(districtCollection).addToMap(map);
    }

    function generateRandomGeoObjects(count, map) {
      const moscowPolygon = moscowJson.features[0];

      const myGeoObjects = new ymaps.GeoObjectCollection({}, {
        preset: "islands#redCircleIcon",
        strokeWidth: 4,
        geodesic: true
      });


      for (let i = 0; i < count; i++) {
        const placemark = new ymaps.Placemark(randomPositionInPolygon(moscowPolygon))
        const placemarkCoords = placemark.geometry.getCoordinates()
        const polygon = state.districtZones.searchContaining(placemarkCoords).get(0)

        // const polygon = ymaps.geoQuery(state.objectManager.objects).searchContaining(placemarkCoords).get(0)
        if(polygon){
          if(state.coordsByRegions[polygon.options.get('id')]){
            state.coordsByRegions[polygon.options.get('id')].push(placemark)
          }else{
            state.coordsByRegions[polygon.options.get('id')] = [placemark]
          }
          myGeoObjects.add(placemark);
        }

        // const geoObjectWithRandomCoords = new ymaps.GeoObject({
        //   geometry: {
        //     type: "Point",
        //     coordinates: randomPositionInPolygon(moscowPolygon),
        //   },
        // });
      }

      const coordsByRegionsCount = {}

      for(const key in state.coordsByRegions){
        coordsByRegionsCount[key] = state.coordsByRegions[key].length
      }

      state.coordsByRegionsCount = coordsByRegionsCount
      
      map.geoObjects.add(myGeoObjects);

      return myGeoObjects;
    }
    
    function regenerateCoords(){
      state.coordsByRegions = new Array(moscowDistrictsJson.features.length)
      state.coordsByRegionsCount = {}
      state.activeRegions = []
      state.activeRegion = null
      state.geoObjectCollection.removeAll()
      state.geoObjectCollection = generateRandomGeoObjects(RANDOM_GEO_OBJECTS_COUNT, state.map);
    }

    function toggleRegion(region){
      if(state.activeRegions.includes(region.id)){
        state.coordsByRegions[region.id].forEach(p => {
          p.options.set('visible', true)
        })
        const idx = state.activeRegions.indexOf(region.id)
        if(idx !== -1){
          state.activeRegions.splice(idx, 1)
        }
      }else{
        state.coordsByRegions[region.id].forEach(p => {
          p.options.set('visible', false)
        })
        state.activeRegions.push(region.id)
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
  <div class="page">
    <div id="map" class="map" />
    <button class="generate-random-btn" @click="regenerateCoords">
      Перегенерировать координаты
    </button>
    <div class="regions">
      <div class="regions__item" v-for="(item,idx) of moscowDistrictsJson.features" :key="idx" @click="toggleRegion(item)">
      {{item.properties.description}}
      {{state.coordsByRegionsCount[item.id]}}
      </div>
    </div>
  </div>
</template>

<style lang="scss">
.map {
  height: 500px;
  width: 500px;
}
</style>