<template>
  <div>
    <h2
      v-if="this.chosenSuggestionMessage.length > 0"
    >{{this.chosenSuggestionMessage[0] + this.chosenLocation.name + this.chosenSuggestionMessage[1]}}</h2>
    <div id="map"></div>
  </div>
</template>

<script>
export default {
  methods: {
    createMarker(place) {
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

    getCurrentPosition() {
      let pos;
      if (this.$route.query.lat && this.$route.query.lng) {
        pos = {
          lat: parseFloat(this.$route.query.lat),
          lng: parseFloat(this.$route.query.lng)
        };
      } else {
        pos = {
          lat: position.coords.latitude,
          lng: position.coords.longitude
        };
        navigator.geolocation.getCurrentPosition(
          position => {
            pos = {
              lat: position.coords.latitude,
              lng: position.coords.longitude
            };
          },
          error => {}
        );
      }
      this.map.setCenter(pos);
      this.service.nearbySearch(
        {
          location: pos,
          radius: this.radius,
          type: this.type
        },
        (results, status) => {
          if (status === google.maps.places.PlacesServiceStatus.OK) {
            this.markers.length = 0;
            this.locations.length = 0;
            this.infowindows.length = 0;
            for (let i = 0; i < results.length; i++) {
              this.markers.push(this.createMarker(results[i]));
              this.locations.push(results[i]);
            }
            this.chooseFromLocations();
            this.chooseFromMessages();
          }
        }
      );
    },
    chooseFromLocations() {
      let index = Math.floor(Math.random() * this.locations.length);
      this.chosenLocation = this.locations[index];
      this.chosenMarker = this.markers[index];
      this.chosenInfowindow = new google.maps.InfoWindow();
      this.chosenInfowindow.setContent(this.chosenLocation.name);
      this.chosenInfowindow.open(this.map, this.chosenMarker);
    },
    chooseFromMessages() {
      let index = Math.floor(Math.random() * this.suggestionMessages.length);
      this.chosenSuggestionMessage = this.suggestionMessages[index];
    }
  },
  data() {
    return {
      map: null,
      service: null,
      markers: [],
      locations: [],
      infowindows: [],
      radius: 500,
      zoom: 15,
      type: ["restaurant"],
      chosenLocation: {},
      chosenMarker: {},
      chosenInfowindow: {},
      suggestionMessages: [
        ["Hey, you should give ", " a try!"],
        ["Why not, ", "?"],
        ["I heard ", " serves good food."],
        ["Have you tried ", "?"],
        ["And our lucky winner is: ", "."]
      ],
      chosenSuggestionMessage: []
    };
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
    this.infowindow = new google.maps.InfoWindow();
    // Get current position.
    this.getCurrentPosition();
  }
};
</script>

<style>
#map {
  height: 400px;
  width: 100%;
}
</style>

