<template>
  <div id="map"></div>
</template>

<script>
export default {
  mounted() {
    const googleMap = document.getElementById("map");
    const options = {
      zoom: 14
    };
    const map = new google.maps.Map(googleMap, options);
    navigator.geolocation.getCurrentPosition(
      position => {
        let pos = {
          lat: position.coords.latitude,
          lng: position.coords.longitude
        };
        map.setCenter(pos);
      },
      error => {}
    );
  }
};
</script>

<style>
#map {
  height: 400px;
  width: 100%;
}
</style>

