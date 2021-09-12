<template>
    <l-map ref="map" :center="[0,0]" :zoom="2" style="z-index:5; height:60vh">
        <l-geo-json ref="geojson" v-if="show_geoJson" :geojson="geojson_data" :options="geojson_options">
        </l-geo-json>
        <l-polygon ref="polygon" v-if="show_polygon" :style="polygon_options" :lat-lngs="polygon_data.features[0].geometry.coordinates">
        </l-polygon>
        <l-tile-layer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" layer-type="base" name="OpenStreetMap" :max-zoom="10" />
    </l-map>
    <div>
        Polygon radius (decimal degrees) = 1
    </div>
    <div>
        Polygon Count <input v-model="polygon_count_input" type="text" v-on:input="updatePolygonCount" name="polygon_count_input">
        <button @click="initPolygonGeneration(polygon_count_input)">Generate Polygon</button>
    </div>
    <div>
        r (between 0 and 1) = {{r_value}} <input v-model="r_value_input" type="text" v-on:input="updateRValue" name="r_value" >
    </div>
    <div>
      Skip last vertex <input v-model="skip_last_vertex" type="checkbox" name="skip_last_vertex"> {{skip_last_vertex}}
    </div>
    <div>
        Point Count <input v-model="point_count_input" type="text" v-on:input="updatepointCount" name="point_count_input">
        <button @click="initpointGeneration(point_count_input)">Generate Points</button>
    </div>
</template>
<script>
import { onBeforeMount, ref, watch } from 'vue';
import "leaflet/dist/leaflet.css"
import { LGeoJson, LMap, LPolygon, LTileLayer } from "@vue-leaflet/vue-leaflet";
export default {
    components: {
        LGeoJson,
        LMap,
        LPolygon,
        LTileLayer,
    },
    setup() {
        const show_polygon = ref(true);
        const show_geoJson = ref(true);
        const r_value_input = ref('');
        const r_value = ref(0.5);
        const updateRValue = () => {
            if (r_value_input.value < 1 && r_value_input.value > 0) {
                r_value.value = r_value_input.value
            } else {
              return console.log('r is out of range')
            }
        }
        const polygon_options = {
            //can't use leaflet methods in here
            style: {
                "color": "#34d399",
                "weight": 2,
                "opacity": 0.65
            },
        };
        const polygon_data = ref({

            type: "FeatureCollection",

            features: [{
                "type": "Feature",
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
                show_polygon.value = !show_polygon.value;
                show_geoJson.value = !show_geoJson.value;
                polygon_data.value.features[0].geometry.coordinates = []
                return generateGeoPolygon(vertex_total);
            }
        }
        const generateGeoPolygon = (vertex_total, position = 1) => {
            if (position > vertex_total) {
                return
            }
            console.log('generateGeoPolygon');
            let angle = (360 / vertex_total) * position;

            polygon_data.value.features[0].geometry.coordinates.push(
                [
                    20 *
                    (Math.round(
                            (Math.sin(angle * (Math.PI / 180)) + Number.EPSILON) * 1000
                        ) /
                        1000),
                    20 *
                    (Math.round(
                            (Math.cos(angle * (Math.PI / 180)) + Number.EPSILON) * 1000
                        ) /
                        1000)
                ]);
            // console.log(polygon_data.value.features[0].geometry.coordinates);
            return generateGeoPolygon(vertex_total, position + 1);
        }
        const polygon_count_input = ref('');
        const polygon_count = ref('');


        var geojson_data = ref({

            type: "FeatureCollection",

            features: [{
                "type": "Feature",
                "geometry": {
                    "type": "Point",
                    "coordinates": [0, 0],
                }
            }],
        });
        var geojson_options = {
            //can't use leaflet methods in here
            style: {
                "color": "#D21F3C",
                "weight": 1,
                "opacity": 0.35
            },
        };
        const point_count_input = ref('');
        const point_count = ref('');
        const points = ref([]);

        const initpointGeneration = (point_total) => {
            if (polygon_count_input.value < 3) {
                return
            }
            points.value = [];
            show_polygon.value = !show_polygon.value;
            show_geoJson.value = false;
            return (generatePoints(point_total))
        }
        const generatePoints = (point_total, point_index = 0, last_vertex=-1) => {
            if (point_index > point_total) {
                console.log('done')
                return addToGeoJson()
            }
            if (point_index === 0) {
                points.value.push([0, 0])
                return (generatePoints(point_total, point_index + 1, 0))
            }
            //random integer from 0 to index length of polygon
            let vertex_random = Math.floor(Math.random() * polygon_count_input.value);
            if (skip_last_vertex.value && vertex_random === last_vertex) {
              return generatePoints(point_total, point_index, last_vertex)
            }
            points.value.push([
                points.value[point_index - 1][0] + ( r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][0] - points.value[point_index - 1][0])),
                points.value[point_index - 1][1] + ( r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][1] - points.value[point_index - 1][1]))
            ])
            return (generatePoints(point_total, point_index + 1, vertex_random))
        }
        const addToGeoJson = async () => {
            geojson_data.value.features = [];
            return points.value.map(point => {
                geojson_data.value.features.push({
                    "type": "Feature",
                    "geometry": {
                        "type": "Point",
                        "coordinates": [point[0], point[1]],
                    }
                });
            });
        }

        const skip_last_vertex = ref('true');

        watch([polygon_data.value, geojson_data.value], async () => {
            console.log('watched');
            show_polygon.value = !show_polygon.value;
            console.log(show_geoJson.value)
            if (show_geoJson.value === false) {
                const { circleMarker } = await import("leaflet/dist/leaflet-src.esm");
                geojson_options.pointToLayer = (feature, latLng) => circleMarker(latLng, { radius: 0.5 });
                return show_geoJson.value = true;
            }
            console.log(show_polygon.value)
            // initPolygonGeneration(polygon_count.value);
        });

        onBeforeMount(async () => {
            const { circleMarker } = await import("leaflet/dist/leaflet-src.esm");
            geojson_options.pointToLayer = (feature, latLng) => circleMarker(latLng, { radius: .5 });
            //initialize map
            ref.mapIsReady = true;
        })
        return {
            addToGeoJson,
            geojson_data,
            geojson_options,
            initpointGeneration,
            initPolygonGeneration,
            polygon_data,
            polygon_options,
            point_count,
            point_count_input,
            points,
            polygon_count,
            polygon_count_input,
            r_value,
            r_value_input,
            show_geoJson,
            show_polygon,
            skip_last_vertex,
            updateRValue,
        }
    }
}
</script>