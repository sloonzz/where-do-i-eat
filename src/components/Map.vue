<template>
  <div>
    <transition name="entrance">
      <div v-show="!loading">
        <h2
          v-if="this.chosenSuggestionMessage.length > 0"
        >{{this.chosenSuggestionMessage[0] + this.chosenLocation.name + this.chosenSuggestionMessage[1]}}</h2>
        <h3
          v-if="this.chosenDistanceMessage.length > 0 && this.distanceToFoodPlace > 0"
        >{{this.chosenDistanceMessage[0] + this.distanceToFoodPlace + " meters" + this.chosenDistanceMessage[1]}}</h3>
        <h4 v-if="this.chosenLocation.rating">{{this.chosenLocation.rating}} / 5.0</h4>
        <input id="pac-input" class="controls" type="text" placeholder="Search Box">
        <div id="map"></div>
        <br>
        <button
          @click.prevent="pickFoodPlaceWithDelay(currentPosition, 200)"
          class="btn"
        >Pick again!</button>
      </div>
    </transition>
  </div>
</template>

<script>
export default {
  methods: {
    initializeAutoComplete() {
      // Create the search box and link it to the UI element.
      let input = document.getElementById("pac-input");
      let searchBox = new google.maps.places.SearchBox(input);
      this.map.controls[google.maps.ControlPosition.TOP_LEFT].push(input);
      let vm = this;
      // // Bias the SearchBox results towards current map's viewport.
      // this.map.addListener("bounds_changed", function() {
      //   searchBox.setBounds(this.map.getBounds());
      // });

      // Listen for the event fired when the user selects a prediction and retrieve
      // more details for that place.
      searchBox.addListener("places_changed", function() {
        let places = searchBox.getPlaces();
        $router.push({
          path: "/",
          query: {
            lat: places[0].geometry.location.lat(),
            lng: places[0].geometry.location.lng()
          }
        });
        vm.initializePositionThenPick();
        if (places.length == 0) {
          return;
        }
      });
    },
    pickFoodPlaceWithDelay(pos, delay) {
      this.loading = true;
      setTimeout(() => {
        this.chooseFromLocations();
        this.chooseFromMessages();
        this.chooseFromDistanceMessages();
        this.loading = false;
      }, delay);
    },
    createMarkerOnPlace(place) {
      let placeLoc = place.geometry.location;
      let marker = new google.maps.Marker({
        map: this.map,
        position: place.geometry.location
      });
      let infowindow = new google.maps.InfoWindow();
      let vm = this;
      marker.addListener("click", function() {
        infowindow.setContent(place.name);
        infowindow.open(this.map, marker);
        $router.push({
          path: "/",
          query: {
            lat: marker.getPosition().lat(),
            lng: marker.getPosition().lng()
          }
        });
        this.map.setCenter({
          lat: parseFloat(vm.$route.query.lat),
          lng: parseFloat(vm.$route.query.lng)
        });
      });
      return marker;
    },
    createMarkerOnPosition(position) {
      if (Object.keys(this.currentLocationMarker).length) {
        this.currentLocationMarker.setMap(null);
      }
      this.currentLocationMarker = new google.maps.Marker({
        map: this.map,
        position: position
      });
      if (Object.keys(this.currentLocationInfoWindow).length) {
        this.currentLocationInfoWindow.close();
      }
      this.currentLocationInfoWindow = new google.maps.InfoWindow();
      let vm = this;

      this.currentLocationInfoWindow.setContent("You are HERE.");
      this.currentLocationInfoWindow.open(this.map, this.currentLocationMarker);
    },
    navGetCurrentPosition(options) {
      return new Promise((resolve, reject) => {
        navigator.geolocation.getCurrentPosition(resolve, reject, options);
      });
    },
    searchNearbyThenPickFoodPlace(pos) {
      let vm = this;
      vm.loading = true;
      return new Promise((resolve, reject) => {
        vm.map.setCenter(pos);
        vm.service.nearbySearch(
          {
            location: pos,
            radius: vm.radius,
            type: vm.type
          },
          (results, status) => {
            if (status === google.maps.places.PlacesServiceStatus.OK) {
              vm.markers.length = 0;
              vm.locations.length = 0;
              for (let i = 0; i < results.length; i++) {
                vm.markers.push(vm.createMarkerOnPlace(results[i]));
                vm.locations.push(results[i]);
              }
              vm.chooseFromLocations();
              vm.chooseFromMessages();
              vm.chooseFromDistanceMessages();
              vm.loading = false;
              resolve();
            } else {
              reject();
            }
          }
        );
      });
    },
    initializePositionThenPick() {
      let pos;
      let vm = this;
      this.markers.forEach(marker => {
        marker.setMap(null);
      });
      this.markers = [];
      return new Promise((resolve, reject) => {
        this.navGetCurrentPosition()
          .then(position => {
            if (vm.$route.query.lat && vm.$route.query.lng) {
              pos = {
                lat: parseFloat(vm.$route.query.lat),
                lng: parseFloat(vm.$route.query.lng)
              };
            } else {
              pos = {
                lat: position.coords.latitude,
                lng: position.coords.longitude
              };
            }
            resolve();
          })
          .then(() => {
            vm.searchNearbyThenPickFoodPlace(pos).then(() => {
              vm.createMarkerOnPosition(pos);
              vm.currentPosition = pos;
            });
          })
          .catch(error => {
            console.log(error);
            reject(error);
          });
      });
    },
    chooseFromLocations() {
      let index = Math.floor(Math.random() * this.locations.length);
      this.chosenLocation = this.locations[index];
      this.chosenMarker = this.markers[index];
      this.map.setCenter(this.chosenLocation.geometry.location);
      this.infowindows.forEach(infowindow => {
        infowindow.close();
      });
      this.chosenInfowindow = new google.maps.InfoWindow();
      this.chosenInfowindow.setContent(this.chosenLocation.name);
      this.chosenInfowindow.open(this.map, this.chosenMarker);
      this.infowindows.push(this.chosenInfowindow);
    },
    chooseFromMessages() {
      let index = Math.floor(Math.random() * this.suggestionMessages.length);
      this.chosenSuggestionMessage = this.suggestionMessages[index];
    },
    chooseFromDistanceMessages() {
      let index = Math.floor(Math.random() * this.distanceMessages.length);
      this.chosenDistanceMessage = this.distanceMessages[index];
    }
  },
  data() {
    return {
      map: null,
      service: null,
      markers: [],
      locations: [],
      infowindows: [],
      radius: 5000,
      zoom: 15,
      type: ["restaurant"],
      currentPosition: {},
      chosenLocation: {},
      chosenMarker: {},
      chosenInfowindow: {},
      currentLocationMarker: {},
      currentLocationInfoWindow: {},
      suggestionMessages: [
        ["Hey, you should give ", " a try!"],
        ["Why not ", "?"],
        ["I heard ", " serves good food."],
        ["Have you tried ", "?"],
        ["And our lucky winner is: ", "."]
      ],
      chosenSuggestionMessage: [],
      distanceMessages: [
        ["", " isn't that far!"],
        ["It's only ", " away from you!"],
        ["Care to drive for ", "?"]
      ],
      chosenDistanceMessage: [],
      loading: true,
      pinpointingLocation: false
    };
  },
  computed: {
    distanceToFoodPlace: function() {
      let vm = this;
      return google.maps.geometry.spherical
        .computeDistanceBetween(
          new google.maps.LatLng(
            this.currentPosition.lat,
            this.currentPosition.lng
          ),
          new google.maps.LatLng(
            this.chosenLocation.geometry.location.lat(),
            this.chosenLocation.geometry.location.lng()
          )
        )
        .toFixed(0);
      // return "HEY";
    }
  },
  mounted() {
    const googleMap = document.getElementById("map");
    const options = {
      zoom: this.zoom
    };
    // Initialize map.
    this.map = new google.maps.Map(googleMap, options);
    // Initialize places service.
    this.service = new google.maps.places.PlacesService(this.map);
    // Get current position.
    this.initializePositionThenPick().then(this.initializeAutoComplete());
  }
};
</script>

<style>
#map {
  height: 400px;
  width: 100%;
}

.entrance-enter,
.entrance-leave-to {
  opacity: 0;
}

.entrance-enter-active,
.entrance-leave-active {
  transition: opacity 200ms ease-out;
}

.entrance-enter-to,
.entrance-leave {
  opacity: 1;
}
</style>
