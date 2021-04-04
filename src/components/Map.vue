<template>
  <div class="map-parent">
    <input
      id="start"
      class="controls"
      type="text"
      placeholder="Адрес отправки"
    />

    <input
      id="end"
      class="controls"
      type="text"
      placeholder="Адрес доставки"
    />

      <input 
      type="submit" 
      class="controls btn-send"
       @click="checkInputsValue" 
       />

    <div :id="mapId" class="map">
      
    </div>
  </div>
</template>

<script>
import { Loader } from "google-maps";
import AutocompleteDirectionsHandler from "../scripts/AutocompleteDirectionsHandler.js";

export default {
  data() {
    return {
      mapId: "myMap",
      google: {},
      inputValue: "",
      marker: null,
      map: null,
      icon: {},
      directionsService: null,
      directionsDisplay: null,
      poly2: null,
      polyline: null,
      startLocation: null,
      endLocation: null,
      step: 1,
      tick: 5,
    };
  },
  methods: {
    load() {
      return Loader;
    },
    
    async initMap() {
      const options = { libraries: ["drawing", "places", "geometry"] };
      const Loader = this.load();
      const loader = new Loader(
        "AIzaSyDDj5TMEKIWC1X4UcOb6-ylDC3ilCRr9RM",
        options
      );

      this.google = await loader.load();
      const self = this;
      this.google.maps.LatLng.prototype.distanceFrom = function (newLatLng) {
        const EarthRadiusMeters = 6378137.0; // meters
        const lat1 = this.lat();
        const lon1 = this.lng();
        const lat2 = newLatLng.lat();
        const lon2 = newLatLng.lng();
        const dLat = ((lat2 - lat1) * Math.PI) / 180;
        const dLon = ((lon2 - lon1) * Math.PI) / 180;
        const a =
          Math.sin(dLat / 2) * Math.sin(dLat / 2) +
          Math.cos((lat1 * Math.PI) / 180) *
            Math.cos((lat2 * Math.PI) / 180) *
            Math.sin(dLon / 2) *
            Math.sin(dLon / 2);
        const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        const d = EarthRadiusMeters * c;
        return d;
      };

      // === A method which returns the length of a path in metres ===
      this.google.maps.Polygon.prototype.Distance = function () {
        let dist = 0;
        for (let i = 1; i < this.getPath().getLength(); i++) {
          dist += this.getPath()
            .getAt(i)
            .distanceFrom(this.getPath().getAt(i - 1));
        }
        return dist;
      };

      // === A method which returns a GLatLng of a point a given distance along the path ===
      // === Returns null if the path is shorter than the specified distance ===
      this.google.maps.Polygon.prototype.GetPointAtDistance = function (
        metres = 1
      ) {
        // some awkward special cases
        if (!metres) metres = 0;
        if (metres == 0) return this.getPath().getAt(0);
        if (metres < 0) return null;
        if (this.getPath().getLength() < 2) return null;
        let dist = 0;
        let olddist = 0;
        for (var i = 1; i < this.getPath().getLength() && dist < metres; i++) {
          olddist = dist;
          dist += this.getPath()
            .getAt(i)
            .distanceFrom(this.getPath().getAt(i - 1));
        }
        if (dist < metres) {
          return null;
        }
        let p1 = this.getPath().getAt(i - 2);
        let p2 = this.getPath().getAt(i - 1);
        let m = (metres - olddist) / (dist - olddist);
        const outLatLong = new self.google.maps.LatLng(
          p1.lat() + (p2.lat() - p1.lat()) * m,
          p1.lng() + (p2.lng() - p1.lng()) * m
        );
        return outLatLong;
      };

      // === A method which returns an array of GLatLngs of points a given interval along the path ===
      this.google.maps.Polygon.prototype.GetPointsAtDistance = function (
        metres
      ) {
        const next = metres;
        const points = [];
        if (metres <= 0) return points;
        let dist = 0;
        let olddist = 0;
        for (let i = 1; i < this.getPath().getLength(); i++) {
          olddist = dist;
          dist += this.getPath()
            .getAt(i)
            .distanceFrom(this.getPath().getAt(i - 1));
          while (dist > next) {
            p1 = this.getPath().getAt(i - 1);
            p2 = this.getPath().getAt(i);
            m = (next - olddist) / (dist - olddist);
            points.push(
              new self.google.maps.LatLng(
                p1.lat() + (p2.lat() - p1.lat()) * m,
                p1.lng() + (p2.lng() - p1.lng()) * m
              )
            );
            next += metres;
          }
        }
        return points;
      };

      // === A method which returns the Vertex number at a given distance along the path ===
      // === Returns null if the path is shorter than the specified distance ===
      this.google.maps.Polygon.prototype.GetIndexAtDistance = function (
        metres
      ) {
        if (metres == 0) return this.getPath().getAt(0);
        if (metres < 0) return null;
        let dist = 0;
        let olddist = 0;
        for (var i = 1; i < this.getPath().getLength() && dist < metres; i++) {
          olddist = dist;
          dist += this.getPath()
            .getAt(i)
            .distanceFrom(this.getPath().getAt(i - 1));
        }
        if (dist < metres) {
          return null;
        }
        return i;
      };
      // === Copy all the above functions to GPolyline ===
      this.google.maps.Polyline.prototype.Distance = this.google.maps.Polygon.prototype.Distance;
      this.google.maps.Polyline.prototype.GetPointAtDistance = this.google.maps.Polygon.prototype.GetPointAtDistance;
      this.google.maps.Polyline.prototype.GetPointsAtDistance = this.google.maps.Polygon.prototype.GetPointsAtDistance;
      this.google.maps.Polyline.prototype.GetIndexAtDistance = this.google.maps.Polygon.prototype.GetIndexAtDistance;

      this.map = new this.google.maps.Map(document.getElementById(this.mapId), {
        center: { lat: 43.23713, lng: 76.9152 },
        zoom: 16,
        mapTypeId: google.maps.MapTypeId.ROADMAP,
      });

      new AutocompleteDirectionsHandler(this.map);
      this.createMarker();

      this.directionsService = new this.google.maps.DirectionsService();
      this.directionsDisplay = new google.maps.DirectionsRenderer({
        map: this.map,
      });

      this.polyline = new this.google.maps.Polyline({
        path: [],
        strokeColor: "#FF0000",
        strokeWeight: 3,
      });

      this.poly2 = new this.google.maps.Polyline({
        path: [],
        strokeColor: "#FF0000",
        strokeWeight: 3,
      });
    },
    
    checkInputsValue() {
      const startPointValue = document.querySelector('#start').value;
      const endPointValue = document.querySelector('#end').value;

      if (startPointValue && endPointValue) {
        this.calcRoute();
      } else {
        alert('Введите адрес отправки и доставки!');
      }
    },

    calcRoute() {
      if (this.timerHandle) {
        clearTimeout(this.timerHandle);
      }
      if (this.marker) {
        this.marker.setMap(null);
      }
      this.polyline.setMap(null);
      this.poly2.setMap(null);
      this.directionsDisplay.setMap(null);
      this.polyline = new this.google.maps.Polyline({
        path: [],
        strokeColor: "#FF0000",
        strokeWeight: 3,
      });
      this.poly2 = new this.google.maps.Polyline({
        path: [],
        strokeColor: "#FF0000",
        strokeWeight: 3,
      });
      const rendererOptions = {
        map: this.map,
      };
      this.directionsDisplay = new this.google.maps.DirectionsRenderer(
        rendererOptions
      );
      const start = document.getElementById("start").value;
      const end = document.getElementById("end").value;
      const travelMode = google.maps.DirectionsTravelMode.DRIVING;

      const request = {
        origin: start,
        destination: end,
        travelMode: travelMode,
      };
      this.directionsService.route(request, (response, status) => {
        if (status == google.maps.DirectionsStatus.OK) {
          this.directionsDisplay.setDirections(response);

          const bounds = new this.google.maps.LatLngBounds();
          // const route = response.routes[0];
          this.startLocation = new Object();
          this.endLocation = new Object();

          // const path = response.routes[0].overview_path;
          const legs = response.routes[0].legs;
          for (let i = 0; i < legs.length; i++) {
            if (i === 0) {
              this.startLocation.latlng = legs[i].start_location;
              this.startLocation.address = legs[i].start_address;
            }
            this.endLocation.latlng = legs[i].end_location;
            this.endLocation.address = legs[i].end_address;
            var steps = legs[i].steps;
            for (let j = 0; j < steps.length; j++) {
              const nextSegment = steps[j].path;
              for (let k = 0; k < nextSegment.length; k++) {
                this.polyline.getPath().push(nextSegment[k]);
                bounds.extend(nextSegment[k]);
              }
            }
          }
          this.polyline.setMap(this.map);
          this.map.fitBounds(bounds);
          this.map.setZoom(18);
          this.startAnimation();
        }
      });
    },

    createMarker(latlng = { lat: 43.23713, lng: 76.9152 }) {
      const car =
        "M17.402,0H5.643C2.526,0,0,3.467,0,6.584v34.804c0,3.116,2.526,5.644,5.643,5.644h11.759c3.116,0,5.644-2.527,5.644-5.644 V6.584C23.044,3.467,20.518,0,17.402,0z M22.057,14.188v11.665l-2.729,0.351v-4.806L22.057,14.188z M20.625,10.773 c-1.016,3.9-2.219,8.51-2.219,8.51H4.638l-2.222-8.51C2.417,10.773,11.3,7.755,20.625,10.773z M3.748,21.713v4.492l-2.73-0.349 V14.502L3.748,21.713z M1.018,37.938V27.579l2.73,0.343v8.196L1.018,37.938z M2.575,40.882l2.218-3.336h13.771l2.219,3.336H2.575z M19.328,35.805v-7.872l2.729-0.355v10.048L19.328,35.805z";
      this.icon = {
        path: car,
        scale: 0.7,
        strokeColor: "white",
        strokeWeight: 0.1,
        fillOpacity: 1,
        fillColor: "#5e0a59",
        offset: "5%",
        anchor: new google.maps.Point(10, 25),
      };
      this.marker = new google.maps.Marker({
        position: new google.maps.LatLng(latlng.lat, latlng.lng),
        map: this.map,
        icon: this.icon,
      });
    },

    updatePoly(d) {
      if (this.poly2.getPath().getLength() > 20) {
        this.poly2 = new this.google.maps.Polyline([
          this.polyline.getPath().getAt(this.lastVertex - 1),
        ]);
      }

      if (this.polyline.GetIndexAtDistance(d) < this.lastVertex + 2) {
        if (this.poly2.getPath().getLength() > 1) {
          this.poly2.getPath().removeAt(this.poly2.getPath().getLength() - 1);
        }
        this.poly2
          .getPath()
          .insertAt(
            this.poly2.getPath().getLength(),
            this.polyline.GetPointAtDistance(d)
          );
      } else {
        this.poly2
          .getPath()
          .insertAt(this.poly2.getPath().getLength(), this.endLocation.latlng);
      }
    },

    animate(d) {
      if (d > this.eol) {
        this.map.panTo(this.endLocation.latlng);
        this.marker.setPosition(this.endLocation.latlng);
        return;
      }
      const p = this.polyline.GetPointAtDistance(d);
      this.map.panTo(p);
      const lastPosn = this.marker.getPosition();
      this.marker.setPosition(p);
      const heading = this.google.maps.geometry.spherical.computeHeading(
        lastPosn,
        p
      );
      this.icon.rotation = heading;
      this.marker.setIcon(this.icon);
      this.updatePoly(d);
      this.timerHandle = setTimeout(() => {
        this.animate(d + this.step);
      }, this.tick);
    },

    startAnimation() {
      this.eol = this.polyline.Distance();
      this.map.setCenter(this.polyline.getPath().getAt(0));
      this.marker = new this.google.maps.Marker({
        position: this.polyline.getPath().getAt(0),
        map: this.map,
        icon: this.icon,
      });

      this.poly2 = new this.google.maps.Polyline({
        path: [this.polyline.getPath().getAt(0)],
        strokeColor: "#0000FF",
        strokeWeight: 10,
      });
      setTimeout(() => {
        this.animate(50);
      }, 2000);
    },
  },
  mounted() {
    this.initMap();
  },
};
</script>

<style>
html,
body {
  height: 100%;
  margin: 0;
  padding: 0;
}

.map-parrent {
  display: grid;
  justify-content: center;
  grid-template-columns: auto;
  width: 95vw;
  height: 95vh;
}

#myMap {  
  width: 95vw;
  height: 95vh;
  margin: auto;
}


.map
-parent {
  display: grid;
  grid-gap: 20px;
}

.controls {
  margin: 10px;
  padding: 5px;
  border: 1px solid transparent;
  border-radius: 2px 0 0 2px;
  box-sizing: border-box;
  -moz-box-sizing: border-box;
  height: 40px;
  outline: none;
  border: 1px solid rgb(187, 101, 101);
  box-shadow: 0 2px 6px rgba(216, 61, 61, 0.5);
  text-overflow: ellipsis;
  transition: .5s;
  letter-spacing: 2px;
}

.controls:hover {
  border: 1px solid rgb(7, 124, 26);
  background-color: rgb(185, 223, 177);
}

.btn-send {
  margin: auto;
  display: block;
  width: 100%;
  background-color: rgba(194, 125, 154, 0.2);
  border: 1px solid rgba(134, 65, 100, 0.2);
  transition: .5s;
  border-radius: 3px;
}

.btn-send:hover {
  background-color: rgba(10, 124, 26, 0.2);
  letter-spacing: 9px;
}
</style>