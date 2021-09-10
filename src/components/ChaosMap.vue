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
        <button @click="initPolygonGeneration(polygon_count_input)">Generate Polygon</button>
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
                    "type": "Polygon",
                    "coordinates": [],
                }
            }],
        });

        const initPolygonGeneration = (vertex_total) => {
            if (vertex_total < 3) {
                return console.log('too few vertices')
            } else {
                console.log('initPolygonGeneration');
                geojson_data.value.features[0].geometry.coordinates = []
                return generateGeoPolygon(vertex_total);
            }
        }
        const generateGeoPolygon = (vertex_total, position = 1) => {
            if (position > vertex_total) {
                return
            }
            console.log('generateGeoPolygon');
            let angle = (360 / vertex_total) * position;

            geojson_data.value.features[0].geometry.coordinates.push(
                [
                    (Math.round(
                        (Math.sin(angle * (Math.PI / 180)) + Number.EPSILON) * 1000
                    ) /
                    1000) ,
                    (Math.round(
                        (Math.cos(angle * (Math.PI / 180)) + Number.EPSILON) * 1000
                    ) /
                    1000) 
                ]);
            console.log(geojson_data.value.features[0].geometry.coordinates);
            return generateGeoPolygon(vertex_total, position + 1);
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
            // initPolygonGeneration(polygon_count.value);
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
            initPolygonGeneration,
            geojson_data,
            polygon_count,
            polygon_count_input
        }
    }
}
</script>