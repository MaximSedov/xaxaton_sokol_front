<template>
  <div style="height: 100%; width: 100%">
     <Toast position="top-center" />
    <!-- Карта -->
    <l-map v-if="showMap" :zoom="zoom" :center="center" :options="mapOptions" style="height: 100%" @click="addMarker" @update:center="centerUpdate" @update:zoom="zoomUpdate">
      <l-tile-layer :url="url" :attribution="attribution" />

      <!-- Полигоны -->
      <!--  <l-polygon :lat-lngs="polygon.latlngs" :color="polygon.color" :fill="polygon.fill" :className="`polygon`"></l-polygon> -->

      <!-- Маркеры -->
      <l-marker v-for="marker, idx in markers" :key="marker[idx]" :lat-lng="marker" @click="removeMarker(idx)" :icon="icon" />

      <!-- Зоны -->
      <!-- <l-circle
        v-for="(circle, index) in circle"
        v-if="sensors"
        :key="index"
        :lat-lng="circle.center"
        :radius="1000"
        :color="circle.color"
        :fillOpacity="0.5"
        :stroke="false"
        :fillColor="circle.fillColor"
        :className="`sensor`"
      /> -->
      <!-- Маршрут  -->
      <l-polyline
        v-for="(line, idx) in polyline"
        :key="line[idx]"
        :lat-lngs="line.latlngs"
        :color="line.color"
        :className="`routes`"
        @mouseover="showRoute($event)"
        @mouseleave="hideRoute($event)"
        :weight="line.weight"
        @click="selectRoute($event)"
      ></l-polyline>

      <!-- Управление на карте -->
      <l-control position="bottomleft" class="control">
        <div class="p-grid p-nogutter p-ai-center p-mt-2 p-jc-start">
          <div class="p-col-4">
            <Button @click="zoom = currentZoom + mapOptions.zoomSnap" icon="pi pi-plus" class="map-control" :disabled="currentZoom === mapOptions.maxZoom" />
          </div>
        </div>
        <div class="p-grid p-nogutter p-ai-center p-mt-2 p-mb-4 p-jc-start">
          <div class="p-col-4">
            <Button @click="zoom = currentZoom - mapOptions.zoomSnap" icon="pi pi-minus" class="map-control" :disabled="currentZoom === mapOptions.minZoom" />
          </div>
        </div>
        <div class="p-grid p-nogutter p-ai-center p-mt-2 p-jc-start">
          <div class="p-col-4">
            <Button @click="polyline = []; markers = []" icon="pi pi-trash" class="map-control" :disabled="markers.length == 0" />
          </div>
        </div>
        <div class="p-grid p-nogutter p-ai-center p-mt-2 p-mt-2 p-jc-start">
          <div class="p-col-4">
            <ToggleButton v-model="checked" @click="sensors = !sensors" class="map-control" onIcon="pi pi-eye-slash" offIcon="pi pi-eye" />
          </div>
        </div>
        <div class="p-grid p-nogutter p-ai-center p-mt-6">
          <div class="p-col-12">
            <Button @click="openControl()" label="Маршрут" icon="pi pi-map" class="map-control" />
          </div>
        </div>
      </l-control>

      <l-control position="bottomright" class="control">
        <Slider v-model="zoom" orientation="vertical" :step="mapOptions.zoomSnap" :min="mapOptions.minZoom" :max="mapOptions.maxZoom" />
      </l-control>

      <!-- <l-marker :lat-lng="withPopup">
        <l-popup>
          <div @click="innerClick">
            I am a popup
            <p v-show="showParagraph">
              Lorem ipsum dolor sit amet, consectetur adipiscing elit. Quisque
              sed pretium nisl, ut sagittis sapien. Sed vel sollicitudin nisi.
              Donec finibus semper metus id malesuada.
            </p>
          </div>
        </l-popup>
      </l-marker>
      <l-marker :lat-lng="withTooltip">
        <l-tooltip :options="{ permanent: true, interactive: true }">
          <div @click="innerClick">
            I am a tooltip
            <p v-show="showParagraph">
              Lorem ipsum dolor sit amet, consectetur adipiscing elit. Quisque
              sed pretium nisl, ut sagittis sapien. Sed vel sollicitudin nisi.
              Donec finibus semper metus id malesuada.
            </p>
          </div>
        </l-tooltip>
      </l-marker>-->
    </l-map>

    <!-- Окно выбора маршрута -->
    <Dialog :visible.sync="display" position="topleft" @click.stop>
      <template #header>
        <h3>Маршрут</h3>
      </template>
      <div class="routeControl" style="display: flex; justify-content: space-between;">
        <Button ref="focusButton" @click="active = 0" icon="pi pi-map" class="p-button-text" label="Из А в Б" />
        <Button @click="active = 1" icon="pi pi-map" class="p-button-text" label="Прогулка" />
      </div>
      <TabView :activeIndex="active">
        <TabPanel header="Из А в Б">
          <div class="p-grid p-nogutter p-ai-center p-mt-2">
            <div class="p-col-12">
              <AutoComplete @click.stop v-model="countryFrom" :suggestions="filteredCountries" @complete="searchCountry($event)" :field="(item) => item.name + ' ' + item.code" />
            </div>
          </div>
          <div class="p-grid p-nogutter p-ai-center p-mt-2">
            <div class="p-col-12">
              <AutoComplete v-model="countryTo" :suggestions="filteredCountries" @complete="searchCountry($event)" :field="(item) => item.name + ' ' + item.code" />
            </div>
          </div>
        </TabPanel>
        <TabPanel header="Прогулка">
          <div class="p-grid p-nogutter p-ai-center p-mt-2">
            <div class="p-col-12">
              <AutoComplete v-model="countryWalk" :suggestions="filteredCountries" @complete="searchCountry($event)" :field="(item) => item.name + ' ' + item.code" />
            </div>
          </div>
        </TabPanel>
      </TabView>
      <SelectButton class="p-mt-2" style="display: flex;justify-content: start;" v-model="selectedType" :options="types" optionLabel="name" />

      <template #footer>
        <Button v-if="markers.length >=2" @click="finishRoute" label="Завершить" icon="pi pi-times" class="p-button-text" />
        <Button v-if="loadRoute" label="Строим" icon="pi pi-spin pi-spinner" disabled />
        <Button v-if="!loadRoute" @click="findRoutes" label="Проложить" icon="pi pi-map" :disabled="countryFrom == null || countryTo == null"/>
      </template>
    </Dialog>
    <!-- <div style="height: 5%; overflow: auto;">
      <p>First marker is placed at {{ withPopup.lat }}, {{ withPopup.lng }}</p>
      <p>Центр: {{ currentCenter }} зум: {{ currentZoom }}</p>
    </div> -->
  </div>
</template>

<script>
import { latLng } from "leaflet";
const axios = require("axios");
import {
  LMap,
  LTileLayer,
  LMarker,
  LPopup,
  LTooltip,
  LPolygon,
  LCircle,
  LIcon,
  LControl,
  LPolyline
} from "vue2-leaflet";
import LRoutingMachine from "./LRoutingMachine.vue";
import { Icon } from "leaflet";

delete Icon.Default.prototype._getIconUrl;
Icon.Default.mergeOptions({
  iconRetinaUrl: require("leaflet/dist/images/marker-icon-2x.png"),
  iconUrl: require("leaflet/dist/images/marker-icon.png"),
  shadowUrl: require("leaflet/dist/images/marker-shadow.png")
});

export default {
  name: "Example",
  components: {
    LMap,
    LTileLayer,
    LMarker,
    LPopup,
    LTooltip,
    LPolygon,
    LCircle,
    LIcon,
    LRoutingMachine,
    LControl,
    LPolyline
  },
  data() {
    return {
      componentKey: 0,
      sensors: true,
      display: false,
      checked: true,
      active: 0,
      countryTo: null,
      countryFrom: null,
      countryWalk: null,
      distance: null,
      time: null,
      filteredCountries: [],
      loadRoute: false,
      countries: [
        { name: "Красная площадь", code: "AF" },
        { name: "Арбат", code: "AL" },
        { name: "Китай город", code: "DZ" },
        { name: "Парк Зарядье", code: "AD" }
      ],
      selectedType: null,
      types: [
        { name: "Пешком", code: "LND" },
        { name: "Самокат", code: "PRS" },
        { name: "Велосипед", code: "RM" }
      ],
      zoom: 13,
      center: latLng(55.7504461, 37.6174943),
      polygon: {
        latlngs: [
          [55.7957880895471, 37.376684908203096],
          [55.7315234841734, 37.364325289062464],
          [55.697412346005066, 37.39453769140622],
          [55.68267328730198, 37.40827060156247],
          [55.636869056546395, 37.453589205078075],
          [55.594899347399334, 37.4989078085937],
          [55.57156334252638, 37.596411470703096],
          [55.57234143382528, 37.69116855078122],
          [55.631431083789636, 37.817511324218714],
          [55.65084893483958, 37.850470308593714],
          [55.69198281565908, 37.833990816406214],
          [55.7082691348538, 37.846350435546846],
          [55.763281945079946, 37.85321689062496],
          [55.81821691873245, 37.84909701757809],
          [55.896229854864345, 37.7186343710937],
          [55.89854466574232, 37.633490328124964],
          [55.91397319398547, 37.58817172460933],
          [55.91243061851775, 37.54285312109371],
          [55.88388185420568, 37.474188570312464],
          [55.88388185420568, 37.44946933203121],
          [55.87461826417799, 37.411017183593714],
          [55.85222208945518, 37.39453769140622],
          [55.83135882022269, 37.40003085546871]
        ],
        color: "#007bff",
        fill: false
      },
      polyline: [
        {
          latlngs: [],
          color: "gray",
          weight: 6
        }
      ],
      circle: [],
      url: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
      attribution:
        '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors',
      //withPopup: latLng(47.41322, -1.219482),
      //withTooltip: latLng(47.41422, -1.250482),
      currentZoom: 13,
      currentCenter: latLng(55.7504461, 37.6174943),
      showParagraph: false,
      mapOptions: {
        zoomSnap: 0.5,
        minZoom: 10,
        maxZoom: 18
      },
      showMap: true,
      icon: L.icon({
        iconUrl: "path.svg",
        iconSize: [32, 80]
      }),
      markers: []
    };
  },
  methods: {
    zoomUpdate(zoom) {
      this.currentZoom = zoom;
      this.zoom = this.currentZoom;
    },
    centerUpdate(center) {
      this.currentCenter = center;
    },
    addMarker(event) {
      if (this.markers.length < 2) {
        this.markers.push(event.latlng);
        if (this.countryFrom == null) {
          this.countryFrom = event.latlng.lng.toFixed(6) + ',' + event.latlng.lat.toFixed(6)
        }else if (this.countryTo == null) {
          this.countryTo = event.latlng.lng.toFixed(6) + ',' + event.latlng.lat.toFixed(6)
        }
        console.log(event.latlng)
      }

      // console.log(this.waypoints);
      // console.log(this.markers);
    },
    /* Удаляем маркеры */
    removeMarker(id) {
      /* for (let index = 0; index < this.markers.length; index++) {
        if (index == id) {
          this.markers.splice(id, 1);
        }
      } */
    },
    hideSensor() {
      /* Прячем датчики */
      this.sensors = false;
    },
    searchCountry(event) {
      setTimeout(() => {
        if (!event.query.trim().length) {
          this.filteredCountries = [...this.countries];
        } else {
          this.filteredCountries = this.countries.filter(country => {
            return country.name
              .toLowerCase()
              .startsWith(event.query.toLowerCase());
          });
        }
      }, 250);
    },
    openControl() {
      this.display = !this.display;
      /* this.$refs['focusButton'].$el.focus() */
    },
    finishRoute() {
      this.countryTo = null;
      this.countryFrom = null;
      this.countryWalk = null;
      this.polyline = [];
      this.markers = [];
    },
    findRoutes() {
      this.loadRoute = true;
      axios
        .get(
          "http://45.80.71.155:5000/route?from=" + this.countryFrom + "&to=" + this.countryTo
        )
        .then(response => {
          let lastitem = response.data.length - 1;
          this.polyline = [];
          for (let index = 0; index < response.data.length; index++) {
            let color = "gray";
            let weight = 6;
            if (index == 0) {
              color = "#007bff";
              weight = 10;
            } else if (index > 0) {
              color = "gray";
              weight = 6;
            }
            let obj = {
              latlngs: [response.data[index].waypoints],
              color: color,
              weight: weight
            };
            this.polyline.push(obj);
          }
          /* this.markers.push(
            L.latLng(
              response.data[0].waypoints[0].lat,
              response.data[0].waypoints[0].lng
            )
          );
          this.markers.push(
            L.latLng(
              response.data[lastitem].waypoints[
                response.data[lastitem].waypoints.length - 1
              ].lat,
              response.data[lastitem].waypoints[
                response.data[lastitem].waypoints.length - 1
              ].lng
            )
          ); */
          this.loadRoute = false;
        })
        .catch(error => {
          this.$toast.add({severity:'error', summary: 'Ошибка', detail:'Неверные данные', life: 3000});
          this.loadRoute = false;
        });
    },
    selectRoute(event) {
      var layer = event.target;
      layer.bringToFront();
      let wei = layer.options.weight;
      let colorr = layer.options.color;
      let flag = false
      if (wei == 10 && colorr == "#007bff") {
        console.log('зашел')
        layer.setStyle({ weight: 10, color: "green" });
        flag = true
      } else if (wei == 10 && colorr == "green" && flag == false) {
        layer.setStyle({ weight: 6, color: "gray" });
        layer.bringToBack();
      }
      else if (wei == 10 && colorr == "green" && flag == true) {
        layer.setStyle({ weight: 6, color: "#007bff" });
        layer.bringToBack();
      }
      else if (wei == 6 && colorr == "gray") {
        layer.setStyle({ weight: 10, color: "green" });
      }
    },
    showRoute(event) {
      /* var layer = event.target
      layer.setStyle({
        weight: 10
      }); */
    },
    hideRoute(event) {
      /* var layer = event.target
      layer.setStyle({
        weight: 6
      }); */
    }
  },
  mounted() {
    axios
      .get("http://45.67.56.16:5000/zones")
      .then(response => {
        //console.log(response.data.sensors);
        for (let index = 0; index < response.data.sensors.length; index++) {
          let obj = {
            center: response.data.sensors[index].center,
            radius: response.data.sensors[index].radius,
            pollution: response.data.sensors[index].pollution
          };
          this.circle.push(obj);
        }
        /* Красим зоны */
        for (let index = 0; index < this.circle.length; index++) {
          if (
            this.circle[index].pollution >= 0 &&
            this.circle[index].pollution <= 50
          ) {
            this.circle[index].fillColor = "green";
          } else if (
            this.circle[index].pollution > 50 &&
            this.circle[index].pollution <= 100
          ) {
            this.circle[index].fillColor = "yellow";
          } else if (
            this.circle[index].pollution > 100 &&
            this.circle[index].pollution <= 150
          ) {
            this.circle[index].fillColor = "orange";
          } else if (
            this.circle[index].pollution > 150 &&
            this.circle[index].pollution <= 200
          ) {
            this.circle[index].fillColor = "red";
          } else if (
            this.circle[index].pollution > 200 &&
            this.circle[index].pollution <= 300
          ) {
            this.circle[index].fillColor = "#800000";
          } else {
            this.circle[index].fillColor = "black";
          }
        }
      })
      .catch(error => {
        console.log(error);
      });
  }
};
</script>

<style lang="scss">
.leaflet {
  &-top {
    margin-top: 8rem !important;
  }
  &-control-zoom {
    display: none;
  }
  &-routing-container {
    display: none;
  }
  &-bottom {
    z-index: 999 !important;
  }
}

.sensor {
  filter: blur(20px);
  -webkit-filter: blur(20px);
  z-index: 1003 !important;
}
.routes {
  z-index: 1004 !important;
}
.map-control {
  width: 100%;
}

.p {
  &-dialog {
    margin-top: 8rem !important;
    border: none !important;
    z-index: 999999 !important;
    &-content {
      height: 14rem;
    }
    &-footer {
      display: flex;
      justify-content: space-between;
    }
    &-footer button {
      margin: 0 !important;
    }
  }
  &-togglebutton {
    &.p-button {
      &.p-highlight {
        background: #007bff !important;
        border: 1px solid #007bff !important;
        box-shadow: none !important;
      }
      &:not(.p-disabled):not(.p-highlight):hover {
        background: #0062cc !important;
        border: 1px solid #0062cc !important;
        box-shadow: none !important;
      }
    }
  }
  &-tabview {
    &-nav {
      display: none !important;
    }
    &-panels {
      padding: 0 !important;
    }
  }
  &-autocomplete {
    width: 100% !important;
    flex-direction: column !important;
  }
  &-selectbutton {
    .p-button {
      background: #ffffff !important;
      border: 1px solid #ced4da !important;
      color: #007bff !important;
      box-shadow: none !important;
      &:hover {
        background: #007bff !important;
        border: 1px solid #007bff !important;
        color: #ffffff !important;
        box-shadow: none !important;
      }
      &.p-selectbutton .p-button.p-highlight:focus {
        background: #007bff !important;
        border: 1px solid #007bff !important;
        color: #ffffff !important;
        box-shadow: none !important;
      }
    }
  }
}

.routeControl .p-button.p-button-text:enabled:focus {
  background: rgba(0, 123, 255, 0.16);
  color: #007bff;
  border-color: transparent;
}

@media (max-width: 767px) {
  .p-dialog {
    margin-top: 0 !important;
    width: 100%;
    border: none !important;
    margin: 0 !important;
  }
  .control {
    margin-bottom: 6rem !important;
  }
}
</style>