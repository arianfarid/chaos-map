<template>
    <div>
        Show tile layer <input v-model="show_tile_layer" type="checkbox" name="show_tile_layer">
        <button @click="resetPolygon()">Reset Polygon</button>
    </div>
    <l-map ref="map" :center="[0,0]" :zoom="3" style="z-index:5; height:60vh">
        <l-marker v-if="constructing_polygon" :lat-lng="[0,0]" draggable @move="updateCoordinates"></l-marker>
        <l-geo-json ref="geojson" v-if="show_geoJson" :geojson="geojson_data" :options="geojson_options">
        </l-geo-json>
        <l-polygon ref="polygon" v-if="show_polygon" :style="polygon_options" :lat-lngs="polygon_data.features[0].geometry.coordinates">
        </l-polygon>
        <l-tile-layer v-if="show_tile_layer" url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" layer-type="base" name="OpenStreetMap" :max-zoom="10" />
    </l-map>
    <!-- check for unintended side affects -->
    <div v-if="!constructing_polygon">
        Polygon Count <input v-model="polygon_count_input" type="text" v-on:input="updatePolygonCount" name="polygon_count_input">
        <button @click="initPolygonGeneration(polygon_count_input)">Generate Equilangular Polygon</button> OR Construct your own Polygon
        <button @click="constructYourPolygon()">Start</button>
    </div>
    <div v-if="constructing_polygon">
        <button v-on:click="addPointToPolygonConstructor()">Add Point</button>
        <button v-on:click="resetPolygon()">Cancel</button>
    </div>
    <div>
        r (between 0 and 1) = {{r_value}} <input v-model="r_value_input" type="text" v-on:input="updateRValue" name="r_value">
    </div>
    <div>
        Skip last vertex <input v-model="skip_last_vertex" type="checkbox" name="skip_last_vertex"> {{skip_last_vertex}}
    </div>
    <div>
        Cannot be one place away (anti-clockwise) from previously chosen vertex
        <input v-model="skip_anti_clockwise_vertex" type="checkbox" name="skip_anti_clockwise_vertex"> {{skip_anti_clockwise_vertex}}
    </div>
    <div v-if="polygon_data.features[0].geometry.coordinates.length > 3">
        Cannot be {{n_places}} places away (anti-clockwise) from previously chosen vertex
        <input v-model="n_places_input" type="text" v-on:input="updateNPlaces()" name="nPlacesInput">
        <input v-model="n_places_true" type="checkbox" name="skip_n_places_true"> {{n_places_true}}
    </div>
    <div>
        Point Count <input v-model="point_count_input" type="text" v-on:input="updatepointCount" name="point_count_input">
        <button @click="initpointGeneration(point_count_input)">Generate Points</button>
    </div>
</template>
<script>
import { onBeforeMount, ref, watch } from 'vue';
import "leaflet/dist/leaflet.css"
import { LGeoJson, LMap, LMarker, LPolygon, LTileLayer } from "@vue-leaflet/vue-leaflet";
export default {
    components: {
        LGeoJson,
        LMap,
        LMarker,
        LPolygon,
        LTileLayer,
    },
    setup() {
        const show_tile_layer = ref(true);
        const show_polygon = ref(true);
        const show_geoJson = ref(true);
        const skip_last_vertex = ref(false);
        const skip_anti_clockwise_vertex = ref(false);
        const n_places = ref('');
        const n_places_input = ref(2);
        const n_places_true = ref(false);
        const updateNPlaces = () => {
          if (n_places_input.value > 1) {
            return n_places.value = n_places_input.value
          } else {
            return console.log('need larger n_input')
          }
        }
        const r_value_input = ref(0.5);
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
                    25 *
                    (Math.round(
                            (Math.sin(angle * (Math.PI / 180)) + Number.EPSILON) * 1000
                        ) /
                        1000),
                    25 *
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
        const resetPolygon = () => {
            show_polygon.value = false;
            constructing_polygon.value = false;
            return polygon_data.value.features[0].geometry.coordinates = [];
        }

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
                "fillColor": "#D21F3C",
                "weight": 0,
                "opacity": 0.45
            },
        };
        const point_count_input = ref(1000);
        const point_count = ref('');
        const points = ref([]);

        const initpointGeneration = (point_total) => {
            if (polygon_data.value.features[0].geometry.coordinates.length < 3) {
                return console.log('too few vertices to initpointGeneration')
            }
            console.log(point_total)
            points.value = [];
            show_polygon.value = !show_polygon.value;
            show_geoJson.value = false;
            return (generatePoints(point_total))
        }


        //TODO 
        //Make a for loop version, avoid stack overflow?
        const generatePoints = (point_total) => {
            points.value = [
                [0, 0]
            ];
            let last_vertex = -1;
            console.log(point_total)
            for (var i = 1; i < point_total - 1; i++) {
                let vertex_random = Math.floor(Math.random() * polygon_count_input.value);
                let anti_clockwise_from_last_vertex = (n) => {

                    if (last_vertex === -n) { // if first in loop, return nothing
                        return -n //will always prove false later
                    } else if (last_vertex - n < 0) { //if first vertex, go to last vertex
                        return polygon_count_input.value - n
                    } else {
                        return last_vertex - n
                    }
                }

                //anticlockwise should be previous vertex -1
                if (skip_anti_clockwise_vertex.value && vertex_random === anti_clockwise_from_last_vertex(1)) {

                    while (vertex_random === anti_clockwise_from_last_vertex(1)) {
                        vertex_random = Math.floor(Math.random() * polygon_count_input.value);
                    }
                }

                //skip last vertex
                if (skip_last_vertex.value && vertex_random === last_vertex) {
                    while (vertex_random === last_vertex) {
                        vertex_random = Math.floor(Math.random() * polygon_count_input.value);
                    }
                }

                //skip n verted anticlockwise
                if (n_places_input.value && vertex_random === anti_clockwise_from_last_vertex(n_places.value)) {
                    while (vertex_random === anti_clockwise_from_last_vertex(n_places.value)) {
                        vertex_random = Math.floor(Math.random() * polygon_count_input.value);
                    }
                }

                // let new_point = [
                //     points.value[point_index - 1][0] + (r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][1] - points.value[point_index - 1][0])),
                //     points.value[point_index - 1][1] + (r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][0] - points.value[point_index - 1][1]))
                // ]
                let generateNewPoint = () => {
                    return [
                        points.value[i - 1][0] + (r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][1] - points.value[i - 1][0])),
                        points.value[i - 1][1] + (r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][0] - points.value[i - 1][1]))
                    ];
                }
                let new_point = generateNewPoint();
                while (!inside(new_point, polygon_data.value.features[0].geometry.coordinates)) {
                    vertex_random = Math.floor(Math.random() * polygon_count_input.value);
                    new_point = generateNewPoint();
                }
                // console.log()
                points.value.push(new_point)

                last_vertex = vertex_random;
            }
            console.log('addToGeoJson')
            return addToGeoJson()
        }
        //
        //  RECURSIVE VERSION
        //  Causes stack overflow, sadly.
        //
        // const generatePoints = (point_total, point_index = 0, last_vertex = -1) => {
        //     if (point_index > point_total) {
        //         console.log('done')
        //         return addToGeoJson()
        //     }
        //     if (point_index === 0) {
        //         points.value.push([0, 0])
        //         return (generatePoints(point_total, point_index + 1, 0))
        //     }
        //     //random integer from 0 to index length of polygon
        //     let vertex_random = Math.floor(Math.random() * polygon_count_input.value);
        //     //generate vertex adjacent value
        //     let anti_clockwise_vertex = () => {
        //         //if first vertex, go to last vertex
        //         if (last_vertex - 1 < 0) {
        //             return polygon_count_input.value - 1
        //         } else {
        //             return last_vertex - 1
        //         }
        //     }
        //     if (skip_last_vertex.value && vertex_random === last_vertex) {
        //         return generatePoints(point_total, point_index, last_vertex)
        //     }

        //     //anticlockwise should be previous vertex -1
        //     if (skip_anti_clockwise_vertex.value && vertex_random === anti_clockwise_vertex()) {
        //         return generatePoints(point_total, point_index, last_vertex)
        //     }
        //     let new_point = [
        //         points.value[point_index - 1][0] + (r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][1] - points.value[point_index - 1][0])),
        //         points.value[point_index - 1][1] + (r_value.value * (polygon_data.value.features[0].geometry.coordinates[vertex_random][0] - points.value[point_index - 1][1]))
        //     ]

        //     if (!inside(new_point, polygon_data.value.features[0].geometry.coordinates)) {
        //         console.log('not inside polygon')
        //         return generatePoints(point_total, point_index, last_vertex)
        //     }

        //     points.value.push(new_point)
        //     return (generatePoints(point_total, point_index + 1, vertex_random))
        // }
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

        const constructing_polygon = ref(false);
        const GPScoordinates = ref(null);
        const constructYourPolygon = () => {
            constructing_polygon.value = true;
            show_polygon.value = false;
            polygon_count_input.value = 0;
            GPScoordinates.value = { lat: 0, lng: 0 }
        }
        const updateCoordinates = (e) => {
            GPScoordinates.value = e.latlng;
        }
        const addPointToPolygonConstructor = () => {
            // show_polygon.value = false;
            console.log(polygon_count_input.value)
            polygon_count_input.value = polygon_count_input.value + 1;
            return polygon_data.value.features[0].geometry.coordinates.push([GPScoordinates.value.lat, GPScoordinates.value.lng]);
        }

        const inside = (point, vs) => {
            // ray-casting algorithm based on
            // https://wrf.ecse.rpi.edu/Research/Short_Notes/pnpoly.html/pnpoly.html
            // thanks stack overflow!

            var x = point[0],
                y = point[1];

            var inside = false;
            for (var i = 0, j = vs.length - 1; i < vs.length; j = i++) {
                var xi = vs[i][1],
                    yi = vs[i][0];
                var xj = vs[j][1],
                    yj = vs[j][0];

                var intersect = ((yi > y) != (yj > y)) &&
                    (x < (xj - xi) * (y - yi) / (yj - yi) + xi);
                if (intersect) inside = !inside;
            }

            return inside;
        };

        watch([polygon_data.value, geojson_data.value], async () => {
            console.log('watched');
            show_polygon.value = !show_polygon.value;
            // console.log(show_geoJson.value)
            if (show_geoJson.value === false) {
                const { circleMarker } = await import("leaflet/dist/leaflet-src.esm");
                geojson_options.pointToLayer = (feature, latLng) => circleMarker(latLng, { radius: 0.7 });
                return show_geoJson.value = true;
            }
            // console.log(show_polygon.value)
            // initPolygonGeneration(polygon_count.value);
        });

        onBeforeMount(async () => {
            const { circleMarker } = await import("leaflet/dist/leaflet-src.esm");
            geojson_options.pointToLayer = (feature, latLng) => circleMarker(latLng, { radius: 0.7 });
            //initialize map
            ref.mapIsReady = true;
        })
        return {
            addPointToPolygonConstructor,
            addToGeoJson,
            constructing_polygon,
            constructYourPolygon,
            geojson_data,
            geojson_options,
            initpointGeneration,
            initPolygonGeneration,
            n_places,
            n_places_input,
            n_places_true,
            polygon_data,
            polygon_options,
            point_count,
            point_count_input,
            points,
            polygon_count,
            polygon_count_input,
            r_value,
            r_value_input,
            resetPolygon,
            show_geoJson,
            show_polygon,
            show_tile_layer,
            skip_last_vertex,
            skip_anti_clockwise_vertex,
            updateCoordinates,
            updateNPlaces,
            updateRValue,
        }
    }
}
</script>