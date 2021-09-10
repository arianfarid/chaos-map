<template>
    <l-map ref="map" :center="[0,0]" style="z-index:5; height:30vh">
        <l-geo-json ref="geojson" :geojson="geojson_data">
        </l-geo-json>
        <l-tile-layer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" layer-type="base" name="OpenStreetMap" :max-zoom="10" />
    </l-map>
    <div>
        Radius (decimal degrees) = 1
    </div>
    <div>
        Polygon Count <input v-model="polygon_count_input" type="text" v-on:input="updatePolygonCount" name="polygon_count_input">
        <button @click="generateGeoPolygon(polygon_count)">Generate Polygon</button>
    </div>
</template>
<script>
import { onBeforeMount, ref, watch } from 'vue';
import "leaflet/dist/leaflet.css"
import { LMap, LGeoJson, LTileLayer } from "@vue-leaflet/vue-leaflet";
export default {
    components: {
        LMap,
        LGeoJson,
        LTileLayer,
    },
    setup() {
        const geojson_options = {
            //can't use leaflet methods in here
            style: {
                "color": "#34d399",
                "weight": 5,
                "opacity": 0.65
            },
        };
        const geojson_data = ref({

            type: "FeatureCollection",

            features: [{
                "type": "Feature",
                "properties": {
                    "dataType": "lat lng coordinate",
                    "notes": "test point."
                },
                "geometry": {
                    "type": "Point",
                    "coordinates": [0, 0],
                }
            }],
        });
        //broke
        const generateGeoPolygon = (num) => {
            console.log(num);
            if (num < 3) {
                return console.log('too few vertices')
            } else {
                console.log('generateGeoPolygon');
            }
            return
        }
        const polygon_count_input = ref('');
        const polygon_count = ref('');
        // const updatePolygonCount = () => {
        //   polygon_count.value = polygon_count_input.value;
        // }
        // const addPointsToMap = (num) => {
        //   if (num === 0) {
        //     return
        //   }
        //   console.log('add');f
        //   return 
        // }
        //not working
        watch(geojson_data, () => {
            console.log('watched');
            if (typeof polygon_count.value != 'number') {
                return
            }
            // generateGeoPolygon(polygon_count.value);
            return
        });
        onBeforeMount(async () => {
            //style geojson
            const { circleMarker } = await import("leaflet/dist/leaflet-src.esm");
            geojson_options.pointToLayer = (feature, latLng) => circleMarker(latLng, { radius: 8 });

            //initialize map
            ref.mapIsReady = true;
        })
        return {
            generateGeoPolygon,
            geojson_data,
            polygon_count,
            polygon_count_input
        }
    }
}
</script>