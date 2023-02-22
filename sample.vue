<template>
  <div id="wrapper">
    <Loader v-show="!loaded" />
    <!-- Main "inner" page for displaying content -->
    <div v-show="loaded" id="explore-container">

          <div id="explore-map-container">
            <div id="explore-map" ref="exploreMap"></div>
          </div>

    </div>
  </div>
</template>

<script>

export default {
  name: "Explore",
  components: {
    Loader: Loader,
  },
  props: [
    "theme",
    "country",
    "filter",
    "territory",
    "itinerary",
    "subItinerary",
    "location",
    "route",
    "monument",
  ],
  mixins: [language, pageDisplay, truncate],
  data() {
    return {
      loaded: false,

      view: {
        background: true,
        directions: false,
        additional: false,
      },

      // Placeholder data
      mapCenterZoom: 4,
      mapCenterCoordinates: {
        lat: 36.728804,
        lng: 3.101106,
      },

      exploreData: {},
      exploreNext: [],
    };
  },
  computed: {
    // Get current "path" and create URLs for breadcrumb links
    // Number of links will always match URL parameters (no need to validate)
    breadcrumbs: function () {
      let filtered = Object.values(this.current).filter(function (crumb) {
        return crumb !== null;
      });
      let array = [];
      let path = this.$route.path.substring(1).split("/");
      for (let i = 0; i < filtered.length; i++) {
        let slice = path.slice(0, i + 2);
        let joined = slice.join("/");
        array.push([filtered[i], joined]);
      }
      // If on Sub-Itinerary Monument page, change Location URL
      // (Remove Monument section of path)
      if (this.$route.name === "itineraries" && this.monument) {
        if (array.length >= 5) {
          let string = array[array.length - 2][1];
          let end = string.lastIndexOf("/");
          array[array.length - 2][1] = string.substr(0, end);
        }
      }
      return array;
    },
  },
  watch: {
    $route(to, from) {
      if (to.hash && !from.params.monument) {
        const element = document.getElementById(to.hash.replace(/#/, ""));
        element.scrollIntoView({ block: 'start', behavior: 'smooth' });
      } else {
        this.loaded = false;
        this.getData();
      }
    },
  },
  methods: {
    locationBreadcrumb: function (array, index) {
      // If on the Itinerary path
      // Location breadcrumb's link needs to be different
      // (Links to Sub-Itinerary page)
      if (this.$route.name === "itineraries") {
        if (array.length >= 5 && this.monument) {
          if (index === array.length - 2) {
            return true;
          }
        }
      }
      return false;
    },
    getData: function () {
      this.displayPageNavigation = false;
      this.buildURL();

      axios({ method: "GET", url: this.builtURL }).then(
        (result) => {
          this.loaded = true;
          // Check to see if page is invalid
          // If so, return and show error page
          if (result.data === "") {
            return this.$router.push({ name: "error" });
          }
          this.exploreData = result.data;
          let results = result.data;

          this.setMapData(this.exploreData);

          // If "last" flag is false, wait to generate map until after next data is received
          // Else, generate the map with the current data (Monument level)
          if (!this.last) {
            this.getNextData(next, isMonuments);
          } else {
            this.currentLocation = this.exploreData.locationId;
            this.$nextTick(() => {
              this.generateMap(this.createMapArray(this.exploreData));
            });
          }
        },
        (error) => {
          console.error(error);
        }
      );
    },
    getNextData: function (next, get) {
      if (!get) {
        this.processNextData(next);
      } else {
        return axios({ method: "GET", url: next }).then(
          (result) => {
            this.processNextData(result.data.data);
            // Set pagination display if necessary
            if (result.data.meta["last_page"] > 1) {
              this.pageInfo = result.data.meta;
              this.pages = result.data.links;
              this.displayPageNavigation = true;
            }
          },
          (error) => {
            console.error(error);
          }
        );
      }
    },
    updateLocation: function (index) {
      this.generateMap(this.createMapArray(this.exploreNext[index].monuments));
      this.setLocation(index);
    },
    createMapArray: function (data) {
      let mapArray = [];

      let info = {};
      if (Array.isArray(data)) {
        for (let i = 0; i < data.length; i++) {
          if (data[i].geoCoordinates) {
            info = {};
            info.latitude = data[i].geoCoordinates.latitude;
            info.longitude = data[i].geoCoordinates.longitude;
            if (
              (this.subItinerary && !this.monument) ||
              (this.route && !this.monument)
            ) {
              info.name = this.returnValueInLanguage(
                "en",
                data[i].monumentName
              ).value;
              info.id = `m-${data[i].monumentId}`;
              // exploreNext[0] is the only option at this level to get Location name
              info.location = this.exploreNext[0].name;
            } else {
              info.name = data[i].name;
              info.id = data[i].thisId;
            }
            mapArray.push(info);
          } else {
            continue;
          }
        }
      } else {
        if (data.geoCoordinates) {
          info.latitude = data.geoCoordinates.latitude;
          info.longitude = data.geoCoordinates.longitude;
          info.name = data.name;
          info.id = data.thisId;
          mapArray.push(info);
        }
      }

      return mapArray;
    },
    setMapData: function (data) {
      this.mapCenterZoom = Number(data.geoCoordinates.zoom);
      this.mapCenterCoordinates.lat = data.geoCoordinates.latitude;
      this.mapCenterCoordinates.lng = data.geoCoordinates.longitude;
    },
    generateMap: function (array) {
      // Create a new Google Maps object
      this.map = new google.maps.Map(document.getElementById("explore-map"), {
        center: this.mapCenterCoordinates,
        zoom: this.mapCenterZoom,
        mapTypeId: "hybrid",
        styles: [
          {
            featureType: "poi",
            elementType: "labels.text",
            stylers: [
              {
                visibility: "off",
              },
            ],
          },
          {
            featureType: "poi.medical",
            stylers: [
              {
                visibility: "off",
              },
            ],
          },
          {
            featureType: "poi.business",
            stylers: [
              {
                visibility: "off",
              },
            ],
          },
          {
            featureType: "road",
            elementType: "labels.icon",
            stylers: [
              {
                visibility: "off",
              },
            ],
          },
          {
            featureType: "transit",
            stylers: [
              {
                visibility: "off",
              },
            ],
          },
        ],
      });

      let pins = [];
      // Loop through array of (pin) objects
      for (let i = 0; i < array.length; i++) {
        let group = [];
        let match = false;
        // Only run if there is information to compare
        if (array.length > 1) {
          // Loop through to find matches
          // Start from next item in array
          for (let j = i + 1; j < array.length; j++) {
            // If latitude matches, check longitude
            if (array[i].latitude === array[j].latitude) {
              if (array[i].longitude === array[j].longitude) {
                match = true;
                group.push(array[j]);
                // Remove matched item
                array.splice(j, 1);
                // Move j back one after splicing
                j -= 1;
              }
            }
          }
        }
        if (match) {
          // Include initial comparison item in array
          group.unshift(array[i]);
          // Push array of matches into pins array
          pins.push(group);
          // Reset group and match variables
          group = [];
          match = false;
        } else {
          // If no matches, push in pin by itself
          pins.push([array[i]]);
        }
      }

      let infoWindow = new google.maps.InfoWindow();
      let marker, z;
      // Bounds automatically calculates map size, fits all pins in window
      let bounds = new google.maps.LatLngBounds();
      for (z = 0; z < pins.length; z++) {
        marker = new google.maps.Marker({
          // [0] because all coordinates in array[z] are the same
          position: new google.maps.LatLng(
            pins[z][0].latitude,
            pins[z][0].longitude
          ),
          map: this.map,
          icon: require("../assets/pointer.png"),
        });
        bounds.extend(marker.position);
        if (!this.monument) {
          google.maps.event.addListener(
            marker,
            "click",
            ((marker, z) => {
              return () => {
                let text = "";
                // If there is more than one pin with the same coordinates
                if (pins[z].length > 1) {
                  // Create link for every "pin" result for combining display in infoWindow
                  for (let i = 0; i < pins[z].length; i++) {
                    // If Sub-Itinerary level, add Location name
                    if (this.subItinerary) {
                      text =
                        text +
                        `
                      <div>
                        <a href="${this.$route.path}/${pins[z][i].id}">
                          ${pins[z][i].location}: ${pins[z][i].name}
                        </a>
                      </div>
                    `;
                    } else {
                      let path = this.$route.path;
                      // if (this.route) {
                      //   let last = path.lastIndexOf("/");
                      //   path = path.slice(0, last);
                      // }
                      text =
                        text +
                        `
                      <div>
                        <a href="${path}/${pins[z][i].id}">
                          ${pins[z][i].name}
                        </a>
                      </div>
                    `;
                    }
                  }
                } else {
                  if (this.subItinerary) {
                    text = `
                    <div>
                      <a href="${this.$route.path}/${pins[z][0].id}">
                        ${pins[z][0].location}: ${pins[z][0].name}
                      </a>
                    </div>
                  `;
                  } else {
                    let path = this.$route.path;
                    // if (this.route) {
                    //   let last = path.lastIndexOf("/");
                    //   path = path.slice(0, last);
                    // }
                    // [0] because it's the only option
                    text = `
                    <a href="${path}/${pins[z][0].id}">
                    ${pins[z][0].name}
                    </a>
                  `;
                  }
                }
                infoWindow.setContent(text);
                infoWindow.open(this.map, marker);
                this.map.panTo(
                  new google.maps.LatLng(
                    pins[z][0].latitude,
                    pins[z][0].longitude
                  )
                );
              };
            })(marker, z)
          );
        }
      }
      this.map.fitBounds(bounds);
      google.maps.event.addListenerOnce(this.map, "idle", () => {
        if (this.map.getZoom() > this.mapCenterZoom) {
          this.map.setZoom(this.mapCenterZoom);
        }
      });
    },
  },
  created() {
    // Insert citation link when copy/pasting from site
    window.addEventListener("copy", (e) => {
      let selection = document.getSelection();
      let citation = `\n\nSource: [${window.location.href}]`;
      e.clipboardData.setData("text/plain", selection + citation);
      e.preventDefault();
    });
  },
  mounted() {
    this.getData();
  },
};
</script>

<style scoped lang="scss">
.loader-container {
  height: 500px;
}

#explore-container {
  display: flex;
  background-color: white;
  width: 100%;
}

#explore-map-container {
  width: 100%;
  margin-top: 30px;
}
#explore-map-header {
  @include headings;
  @include font-size-3;
  margin-top: 40px;
  margin-bottom: 10px;
}
#explore-map {
  height: 500px;
  width: 100%;
  margin-bottom: 40px;
}
/* IE-only */
@media only screen and (-ms-high-contrast: none), (-ms-high-contrast: active) {
  .explore-next-tiles {
    display: flex;
    flex-wrap: wrap;
  }
  .explore-next-tile {
    width: 32%;
    margin: 5px;
  }
  .explore-next-link {
    top: 0;
    left: 0;
  }
}

@media only screen and (max-width: 1199px) {
  #explore-container {
    flex-direction: column;
  }
  #side-navigation-container {
    width: 100%;
    padding: 40px 40px 0 40px;
  }
  #explore {
    width: 100%;
  }
}

@media only screen and (max-width: 949px) {
  #explore-header {
    @include font-size-3;
  }
  .explore-next-tiles {
    grid-template-columns: repeat(2, 1fr);
  }
  .explore-next-label {
    @include font-size-3;
  }
  /* IE-only */
  @media only screen and (-ms-high-contrast: none),
    (-ms-high-contrast: active) {
    .explore-next-tiles {
      display: flex;
      flex-wrap: wrap;
    }
    .explore-next-tile {
      width: 48%;
      margin: 5px;
    }
  }
}

@media only screen and (max-width: 649px) {
  .explore-next-tile {
    height: 150px;
  }
  .explore-next-label {
    @include font-size-4;
  }
  /* IE-only */
  @media only screen and (-ms-high-contrast: none),
    (-ms-high-contrast: active) {
    .explore-next-tile {
      width: 46%;
    }
  }
}

@media only screen and (max-width: 499px) {
  #side-navigation-container {
    padding: 30px 30px 0 30px;
  }
  #explore {
    padding: 0 30px;
  }
  #explore-header,
  .explore-next-header {
    @include font-size-4;
  }
  #explore-header {
    margin-top: 20px;
  }
  #explore-map-container {
    margin-top: 20px;
  }
  #explore-map {
    height: 400px;
  }
  .explore-next-tiles {
    display: flex;
    flex-direction: column;
    width: 100%;
  }
  .explore-next-tile {
    width: 100%;
  }
  .explore-next-tiles .monument-tiles {
    margin-bottom: 30px;
  }
  /* IE-only */
  @media only screen and (-ms-high-contrast: none),
    (-ms-high-contrast: active) {
    .explore-next-tile {
      width: 100%;
    }
  }
}
</style>
