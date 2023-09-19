<template>
  <div class="map-container">
    <div class="controls">
      <button @click="getUserLocation">Get My Location</button>
      <input
        v-model="searchQuery"
        @keydown.enter="searchLocation"
        placeholder="Enter a location"
      />
      <button @click="searchLocation">Search</button>
    </div>
    <div ref="map" :style="{ width: iframeWidth + 'px', height: iframeHeight + 'px' }"></div>
    <div class="search-results">
      <table>
        <thead>
          <tr>
            <th></th> 
            <th>Address</th>
            <th>Time Zone</th>
            <th>Local Time</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(place, index) in searchResults" :key="index">
            <td>
              <input type="checkbox" v-model="selectedPlaces" :value="place" />
            </td>
            <td>{{ place.formatted_address }}</td>
            <td>{{ place.timeZone }}</td>
            <td>{{ place.localTime }}</td>
          </tr>
        </tbody>
      </table>
      <div class="pagination">
        <button @click="prevPage" :disabled="currentPage === 1">Previous</button>
        <span>{{ currentPage }}</span>
        <button @click="nextPage" :disabled="currentPage === totalPages">Next</button>
      </div>
      <button @click="deleteSelectedPlaces" class="delete-button">Delete Selected Places</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      apiKey: 'AIzaSyDsHFPKNHEH8Ew8yub9Kl1yLsVzB8aO2nY',
      location: { lat: 47.6205, lng: -122.3493 }, 
      iframeWidth: 600,
      iframeHeight: 450,
      searchQuery: '',
      map: null,
      markers: [],
      selectedPlaces: [],
      currentPage: 1,
      pageSize: 10, 
      searchResults: [],
      placeMarkerMap: {},
    };
  },
  computed: {
    mapUrl() {
      return `https://www.google.com/maps/embed/v1/place?key=${this.apiKey}&q=${this.location.lat},${this.location.lng}`;
    },
    totalPages() {
      return Math.ceil(this.markers.length / this.pageSize);
    },
    paginatedResults() {
      const startIndex = (this.currentPage - 1) * this.pageSize;
      const endIndex = startIndex + this.pageSize;
      return this.markers.slice(startIndex, endIndex);
    },
  },
  methods: {
    initMap() {

      this.map = new google.maps.Map(this.$refs.map, {
        center: this.location,
        zoom: 15,
      });
    },
    async getUserLocation() {
      if ('geolocation' in navigator) {
        navigator.geolocation.getCurrentPosition(async (position) => {
          const { latitude, longitude } = position.coords;
          this.location = { lat: latitude, lng: longitude };
          this.updateMap();
          try {
            const timeZoneResponse = await fetch(
              `https://maps.googleapis.com/maps/api/timezone/json?location=${latitude},${longitude}&timestamp=${Math.floor(
                Date.now() / 1000
              )}&key=${this.apiKey}`
            );

            if (timeZoneResponse.ok) {
              const timeZoneData = await timeZoneResponse.json();
              const timeZone = timeZoneData.timeZoneId;
              try {
                const geocodeResponse = await fetch(
                  `https://maps.googleapis.com/maps/api/geocode/json?latlng=${latitude},${longitude}&key=${this.apiKey}`
                );

                if (geocodeResponse.ok) {
                  const geocodeData = await geocodeResponse.json();
                  if (geocodeData.results.length > 0) {
                    const result = geocodeData.results[0];

                    const marker = new google.maps.Marker({
                      position: this.location,
                      map: this.map,
                      title: 'Location',
                    });
                    this.placeMarkerMap[result.formatted_address] = marker;

                    const formattedAddressExists = this.searchResults.some(
                      item => item.formatted_address === result.formatted_address
                    );
                    if (!formattedAddressExists) {
                      this.searchResults.push({
                        formatted_address: result.formatted_address,
                        timeZone: timeZone,
                        localTime: new Date().toLocaleTimeString([], { timeZone: timeZone }),
                      });
                    } else {
                    }
                  }
                }
              } catch (error) {
                console.error('Error reverse geocoding user location:', error);
              }
            }
          } catch (error) {
            console.error('Error getting time zone data:', error);
          }
        }, (error) => {
          console.error('Error getting user location:', error);
        });
      } else {
        console.error('Geolocation is not supported in this browser.');
      }
    },
    async searchLocation() {
    if (this.searchQuery) {
      try {
        const response = await fetch(
          `https://maps.googleapis.com/maps/api/geocode/json?address=${encodeURIComponent(this.searchQuery)}&key=${this.apiKey}`
        );

        if (response.ok) {
          const data = await response.json();
          if (data.results.length > 0) {
            const result = data.results[0];
            const { lat, lng } = result.geometry.location;
            this.location = { lat: lat, lng: lng };
            this.updateMap();

            const timeZoneResponse = await fetch(
              `https://maps.googleapis.com/maps/api/timezone/json?location=${lat},${lng}&timestamp=${Math.floor(
                Date.now() / 1000
              )}&key=${this.apiKey}`
            );
            if (timeZoneResponse.ok) {
              const timeZoneData = await timeZoneResponse.json();
              const timeZone = timeZoneData.timeZoneId;
              const marker = new google.maps.Marker({
                position: this.location,
                map: this.map,
                title: 'Location',
              });
              this.placeMarkerMap[result.formatted_address] = marker;
              const formattedAddressExists = this.searchResults.some(
                      item => item.formatted_address === result.formatted_address
                    );
              if (!formattedAddressExists) {
                this.searchResults.push({
                  formatted_address: result.formatted_address,
                  timeZone: timeZone,
                  localTime: new Date().toLocaleTimeString([], { timeZone: timeZone }),
                });
              } else {
              }
            } else {
              console.error('Error fetching time zone data:', timeZoneResponse.statusText);
            }
          } else {
            console.error('No results found for the search query.');
          }
        } else {
          console.error('Error fetching location data:', response.statusText);
        }
      } catch (error) {
        console.error('Error:', error);
      }
    } else {
      console.error('Search query is empty.');
    }
  },
    updateMap() {
      this.markers.forEach((marker) => {
        marker.setMap(null);
      });
      this.markers = [];

      const marker = new google.maps.Marker({
        position: this.location,
        map: this.map,
        title: 'Location',
      });
      this.markers.push(marker);
      this.map.setCenter(this.location);
    },

    prevPage() {
      if (this.currentPage > 1) {
        this.currentPage--;
      }
    },
    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    },
    deleteSelectedPlaces() {
      this.selectedPlaces.forEach((place) => {
        const index = this.searchResults.indexOf(place);
        if (index !== -1) {
          this.searchResults.splice(index, 1);
          const marker = this.placeMarkerMap[place.formatted_address];
          if (marker) {
            marker.setMap(null);
            delete this.placeMarkerMap[place.formatted_address];
          }
        }
      });

      this.selectedPlaces = [];
    },
  },
  mounted() {
    const script = document.createElement('script');
    script.src = `https://maps.googleapis.com/maps/api/js?key=${this.apiKey}&callback=initMap`;
    script.async = true;
    script.defer = true;
    script.onerror = () => {
      console.error('Error loading Google Maps API.');
    };
    window.initMap = this.initMap;
    document.head.appendChild(script);
    this.getUserLocation();
  },
};
</script>

<style>
.map-container {
  position: relative;
}

.controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 10px;
}

.controls input {
  flex-grow: 1;
  margin-right: 10px;
}

.search-results {
  margin-top: 20px;
}

table {
  width: 100%;
  border-collapse: collapse;
}

table th,
table td {
  padding: 8px;
  border: 1px solid #ddd;
  text-align: left;
}

table th {
  background-color: #f2f2f2;
}

.pagination {
  margin-top: 10px;
  text-align: center;
}

.pagination button {
  margin: 0 5px;
}

.delete-button {
  margin-top: 10px;
}
</style>
