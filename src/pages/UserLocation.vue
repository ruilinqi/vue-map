<template>
  <div>
    <div class="ui center aligned basic segment">
      <button class="ui button sui-button basic primary" @click="acquireCurrentLocation">Acquire Current Location</button>
    </div>
    <div class="search-container ui center aligned basic segment">
      <div class="ui left icon action input">
        <i class="search icon"></i>
        <input type="text" placeholder="Enter location" v-model="searchTerm" @keydown.enter="searchLocation">
      </div>
      <button class="ui button sui-button blue" @click="searchLocation">Search</button>
    </div>
    <div class="map-container">
      <section id="map" ref="map"></section>
    </div>

    <div class="table-container">
      <table class="ui celled striped table">
        <thead>
          <tr>
            <th></th>
            <th>Location</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="place in paginatedPlaces" :key="place.id">
            <td><input type="checkbox" v-model="selectedPlaces" :value="place.id"></td>
            <td>{{ place.name }}</td>
          </tr>
        </tbody>
      </table>
      <button class="ui button blue" @click="deleteSelected">Delete Selected</button>
    </div>

    <div class="ui pagination menu">
      <a class="item" v-for="page in totalPages" :key="page" @click="goToPage(page)">{{ page }}</a>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      searchTerm: "",
      searchedPlaces: [],
      selectedPlaces: [],
      currentPage: 1,
      pageSize: 10,
      map: null,
      markers: [],
    };
  },

  mounted() {
    this.initializeMap();
    this.initializeAutocomplete();
  },


  methods: {
    acquireCurrentLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition((position) => {
          const latitude = position.coords.latitude;
          const longitude = position.coords.longitude;
          this.showLocationOnTheMap(latitude, longitude);
        }, this.handleLocationError);
      } else {
        alert("Geolocation is not supported by this browser.");
      }
    },

    initializeMap() {
      this.map = new google.maps.Map(this.$refs.map, {
        zoom: 15,
        center: new google.maps.LatLng(43.6532, -79.3832),
        mapTypeId: google.maps.MapTypeId.ROADMAP,
      });
    },

    initializeAutocomplete() {
      const autocomplete = new google.maps.places.Autocomplete(this.$refs.autocompleteInput, {
        types: ["geocode"],
      });

      autocomplete.addListener("place_changed", () => {
        const place = autocomplete.getPlace();
        if (place && place.geometry && place.geometry.location) {
          const latitude = place.geometry.location.lat();
          const longitude = place.geometry.location.lng();
          this.showLocationOnTheMap(latitude, longitude);
        }
      });
    },

    showLocationOnTheMap(latitude, longitude) {
      this.clearMarkers();

      const marker = new google.maps.Marker({
        position: new google.maps.LatLng(latitude, longitude),
        map: this.map,
      });
      this.markers.push(marker);

      this.map.setCenter(marker.getPosition());
    },

    searchLocation() {
      if (this.searchTerm.trim() === "") {
        return;
      }

      const query = encodeURIComponent(this.searchTerm);
      const apiKey = "[API KEY]"; // Replace with your Google Maps API key

      const url = `https://maps.googleapis.com/maps/api/place/textsearch/json?query=${query}&key=${apiKey}`;

      axios
        .get(url)
        .then((response) => {
          this.searchedPlaces = response.data.results;
          //this.clearMarkers();
          this.searchedPlaces.forEach((place) => {
            this.addMarker(place);
          });
        })
        .catch((error) => {
          console.error(error);
        });
    },

    addMarker(place) {
      const latitude = place.geometry.location.lat;
      const longitude = place.geometry.location.lng;

      const marker = new google.maps.Marker({
        position: { lat: latitude, lng: longitude },
        map: this.map,
      });

      this.markers.push(marker);
    },

    clearMarkers() {
      this.markers.forEach((marker) => {
        marker.setMap(null);
      });
      this.markers = [];
    },

    deleteSelected() {
      this.searchedPlaces = this.searchedPlaces.filter((place) => !this.selectedPlaces.includes(place.id));
      this.clearMarkers();
      this.selectedPlaces = [];
      this.searchedPlaces.forEach((place) => {
        this.addMarker(place);
      });
    },

    goToPage(page) {
      this.currentPage = page;
    },
  },

  computed: {
    paginatedPlaces() {
      const startIndex = (this.currentPage - 1) * this.pageSize;
      const endIndex = startIndex + this.pageSize;
      return this.searchedPlaces.slice(startIndex, endIndex);
    },

    totalPages() {
      return Math.ceil(this.searchedPlaces.length / this.pageSize);
    },
  },
};
</script>

<style scoped>
.search-container {
  margin-bottom: 20px;
}

.map-container {
  height: 400px;
  margin-bottom: 20px;
}

.table-container {
  margin-bottom: 20px;
}

.ui.button.blue {
  background-color: #336bd3;
  color: white;
}

.ui.pagination.menu {
  margin-top: 20px;
}

#map {
  height: 100%;
  width: 100%;
}
</style>
