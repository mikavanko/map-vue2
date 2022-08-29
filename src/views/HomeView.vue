<script>
import Map from "@/components/Map.vue";

const RANDOM_GEO_OBJECTS_COUNT = 100;

export default {
  components: {
    Map,
  },
  setup() {
    return {
      RANDOM_GEO_OBJECTS_COUNT,
    };
  },
};
</script>
  
<template>
  <div class="page">
    <Map :geoObjectsCount="RANDOM_GEO_OBJECTS_COUNT" />
  </div>
</template>

<style lang="scss">
</style>