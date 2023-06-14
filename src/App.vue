<template>
  <div>
    <h1>Vue Map App</h1>
    <hr />

    <div class="location">
      <button class="current" @click="getCurrentLocation">Get Current Location</button>

      <div class="search">
        <input v-model="searchText" @keyup.enter="searchLocation" placeholder="Search location" />
        <button  @click="searchLocation">Search</button>
      </div>
    </div>
    

    <div id="map">
    </div>

    <div class="table-container">
      <div class="table-button">
        <button @click="deleteSelectedLocations">Delete Selected</button>
        <button @click="deleteAllLocations">Delete All</button>
      </div>
      
      <table>
        <thead>
          <tr>
            <th></th>
            <th>Name</th>
            <th>Latitude</th>
            <th>Longitude</th>
            <th>Time Zone</th>
            <th>Local Time</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="location in paginatedLocations" :key="location.id">
            <td><input type="checkbox" v-model="selectedLocations" :value="location.id" /></td>
            <td>{{ location.name }}</td>
            <td>{{ location.latitude }}</td>
            <td>{{ location.longitude }}</td>
            <td>{{ location.timeZone }}</td>
            <td>{{ location.localTime }}</td>
          </tr>
        </tbody>
      </table>

      <ul class="pagination">
      <li v-for="pageNumber in totalPages" :key="pageNumber" :class="{ active: pageNumber === currentPage }">
        <button @click="currentPage = pageNumber">{{ pageNumber }}</button>
      </li>
    </ul>

    <div>
      <p>Latest Time Zone: {{ locations.length > 0 ? locations[locations.length - 1].timeZone : '' }}</p>
      <p>LatestLocal Time: {{ locations.length > 0 ? locations[locations.length - 1].localTime : '' }}</p>
    </div>
    </div>
    

    
  </div>
</template>

<script>

import { ref, onMounted } from 'vue';
import {toRaw} from 'vue';


export default {
  setup() {
    const mapLoaded = ref(false);

    onMounted(() => {
      const googleMapsScript = document.createElement('script');
      googleMapsScript.src = `https://maps.googleapis.com/maps/api/js?key=AIzaSyDxbW4U5N1f5VoLFQwpGmLq6rmIyubhcUY&callback=initMap`;
      googleMapsScript.defer = true;
      window.initMap = () => {
        this.map = new window.google.maps.Map(document.getElementById('map'), {
          center: { lat: 53, lng:-113 },
          zoom: 10
        });
        mapLoaded.value = true;
      };
      document.head.appendChild(googleMapsScript);
    });

    return {
      mapLoaded
    };
  },
 
  
  data() {
    return {
      searchQuery: '',
      locations: [],
      selectedLocations: [],
      currentPage: 1,
      itemsPerPage: 10,
      markers: [],
      currentLocation: [],
      timeZone: '',
      localTime: ''
    }
  },
  computed: {
    totalPages() {
    return Math.ceil(this.locations.length / this.itemsPerPage);
  },
    paginatedLocations() {
      const reversedLocations = this.locations.slice().reverse();
      const startIndex = (this.currentPage - 1) * this.itemsPerPage;
      const endIndex = startIndex + this.itemsPerPage;
      return reversedLocations.slice(startIndex, endIndex);
    },
  },
  methods: {
    
    getCurrentLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(this.handleLocationSuccess, this.handleLocationError);
        
      } else {
        console.log('Geolocation is not supported by this browser.');
      }
    },



    handleLocationSuccess(position) {
      const latitude = position.coords.latitude;
      const longitude = position.coords.longitude;
      
      const timeZoneUrl = `https://maps.googleapis.com/maps/api/timezone/json?location=${latitude},${longitude}&timestamp=${Math.floor(Date.now() / 1000)}&key=AIzaSyDxbW4U5N1f5VoLFQwpGmLq6rmIyubhcUY`;
      fetch(timeZoneUrl)
        .then(response => response.json())
        .then(data => {
          if (data.status === 'OK') {
            const timeZone = data.timeZoneId;
            const localTime = new Date().toLocaleString('en-US', { timeZone });

            this.locations.push({ id: Date.now(), name: "Current Location", latitude, longitude, timeZone, localTime });
            this.showMap(latitude, longitude);
          }
        })
        .catch(error => {
          console.log('Error retrieving time zone:', error);
        });
    

      this.showMap(latitude, longitude);
    },


    handleLocationError(error) {
      console.log('Error occurred while getting the current location:', error);
    },




    searchLocation() {
      if (this.searchText.trim() === '') {
        return; 
      }

      const geocoder = new google.maps.Geocoder();

      geocoder.geocode({ address: this.searchText }, (results, status) => {
        if (status === google.maps.GeocoderStatus.OK) {
          const result = results[0];
          const latitude = result.geometry.location.lat();
          const longitude = result.geometry.location.lng();
          const locationName = result.formatted_address;
          const timeZoneUrl = `https://maps.googleapis.com/maps/api/timezone/json?location=${latitude},${longitude}&timestamp=${Math.floor(Date.now() / 1000)}&key=AIzaSyDxbW4U5N1f5VoLFQwpGmLq6rmIyubhcUY`;
          fetch(timeZoneUrl)
            .then(response => response.json())
            .then(data => {
              if (data.status === 'OK') {
                const timeZone = data.timeZoneId;
                const localTime = new Date().toLocaleString('en-US', { timeZone });

                this.locations.push({ id: Date.now(), name: locationName, latitude, longitude, timeZone, localTime });
                this.showMap(latitude, longitude);
              }
            })
            .catch(error => {
              console.log('Error retrieving time zone:', error);
            });
        } 
        else {
          console.log('Geocode was not successful for the following reason:', status);
        }

        this.searchText = ''; 
  });
    },




    showMap(latitude, longitude) {
      const map = new google.maps.Map(document.getElementById('map'), {
        center: { lat: latitude, lng: longitude },
        zoom: 12
      });

      this.locations.forEach(location => {
        const marker = new google.maps.Marker({
          position: { lat: location.latitude, lng: location.longitude },
          title: location.name,
          id: location.id
        });
        marker.setMap(map);
        this.markers.push(marker);
        console.log("added")
      });
      
    },


    deleteSelectedLocations() {

      this.locations = this.locations.filter(location => !this.selectedLocations.includes(location.id));
      this.selectedLocations.forEach(selectedId => {
      const markerIndex = this.markers.findIndex(marker => marker.id === selectedId);
      if (markerIndex !== -1) {
        console.log(this.markers[markerIndex])
        toRaw(this.markers[markerIndex]).setMap(null);
        this.markers.splice(markerIndex, 1);
        console.log("deleted")
      }
    });
      this.selectedLocations = [];

    },


    
    deleteAllLocations() {
      this.locations = [];
      this.markers.forEach(marker =>{
        toRaw(marker).setMap(null);
      });
      this.selectedLocations = [];
    }




  }
}
</script>

<style>
h1 {

  font-size: 1.7em;
  font-weight: 1000;
  font-family: Georgia, serif;
}

.location {
  display: flex;
  flex-direction:row;
  justify-content: space-between;
  margin: 20px;
}

button {
  border: solid gray 1.5px;
  font-size: 1em;
}

.search input {
  font-size: 1em;
  margin-right: 10px;
}
.search button {
  border: solid gray 1.5px;
  font-size: 1em;
}

#map {
  margin-bottom:50px ;
  width: 80vw;
  height: 650px;
}

.table-container {
  background-color: rgb(228, 234, 232);
  padding: 20px;
}

.table-button {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  margin-bottom: 10px;
}
table {
  width: 100%;
  border-collapse: collapse;
}

table th, table td {
  padding: 8px;
  border: 1px solid #b2b0b0;
}
.pagination {
  list-style-type: none;
  margin: 0;
  padding: 0;
  display: flex;
  justify-content: center;
}
.pagination button {
  margin-top: 10px;
  padding: 5px 10px;
  border: 1px solid gray;
  background-color: #e2dddd;
  cursor: pointer;
}



</style>