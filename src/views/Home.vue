<template lang="pug">
  #map-container
    .app-toolbar
      b-row.align-items-center.justify-content-between
        b-col(cols="auto")
          b-card.px-3.py-2.mb-0(no-body)
            .d-flex.align-items-center
              v-icon.pointer.mr-3(name="bars", scale="1.5")
              b-form-input#search_box(placeholder="Buscar lugar...", type="search", onclick="this.select()")
              v-icon.pointer.ml-3.d-none.d-md-block(name="print", scale="1.5", v-b-tooltip.hover="'Exportar registros'", @click.native="export_modal = true")
              v-icon.pointer.ml-3(name="compass", scale="1.5", v-if="currentLocation", v-b-tooltip.hover="'Mi ubicación'", @click.native="setCurrentLocation")
        b-col.d-none.d-md-block(cols="auto")
          b-button(:variant="show_heat_map ? 'secondary' : 'primary'", @click="switchHeatmapMarkers") {{ show_heat_map ? 'Quitar' : 'Ver' }} mapa de calor
    #map
    b-modal(size="lg", title="Exportar registros", v-model="export_modal", centered, scrollable, @show="getVisiblePoints", ok-only, ok-title="Cerrar", ok-variant="secondary", hide-header-close)
      b-container(fluid)
        .d-flex.align-items-center.justify-content-between.mb-4
          p.mb-0.text-secondary Se exportarán sólo los reportes visibles en el área del mapa.
          b-button(variant="success", :disabled="!export_items.length") Exportar a csv
        b-table(:items="export_items", outlined, show-empty)
          template(slot="empty")
            p.mb-0.text-secondary No hay registros en el área visible del mapa
</template>

<script>
import Maps from '@/helpers/maps'
import MarkerClusterer from '@google/markerclusterer'
import MapTheme from '@/helpers/map-theme'

const gradient = [
  'rgba(0, 255, 255, 0)',
  'rgba(0, 255, 255, 1)',
  'rgba(0, 191, 255, 1)',
  'rgba(0, 127, 255, 1)',
  'rgba(0, 63, 255, 1)',
  'rgba(0, 0, 255, 1)',
  'rgba(0, 0, 223, 1)',
  'rgba(0, 0, 191, 1)',
  'rgba(0, 0, 159, 1)',
  'rgba(0, 0, 127, 1)',
  'rgba(63, 0, 91, 1)',
  'rgba(127, 0, 63, 1)',
  'rgba(191, 0, 31, 1)',
  'rgba(255, 0, 0, 1)'
]

export default {
  data () {
    return {
      export_modal: false,
      show_heat_map: false,
      map: null,
      heatmap: null,
      locations: [
        {
          position: {
            lat: 19.434208,
            lng: -99.1513664,
          }
        },
        {
          position: {
            lat: 19.4273871,
            lng: -99.1652376
          }
        }
      ],
      markers: [],
      currentLocation: null,
      export_items: []
    }
  },
  methods: {
    switchHeatmapMarkers (force) {
      this.show_heat_map = (typeof force == 'boolean' ? force == true : !this.show_heat_map)

      if (this.show_heat_map) {
        this.heatmap.setMap(this.map)
        // this.setMarkersOnMap(null)
      } else {
        this.heatmap.setMap(null)
        // this.setMarkersOnMap(this.map)
      }
    },
    setMarkersOnMap (map) {
      for (var i = 0; i < this.markers.length; i++) {
        this.markers[i].setMap(map);
      }
    },
    centerOnLocation (location) {
      const geocoder = new google.maps.Geocoder()
      geocoder.geocode(location, (results, status) => {
        if (status !== 'OK' || !results[0]) {
          throw new Error(status);
        }

        this.map.setCenter(results[0].geometry.location);
        if (location.address)
          this.map.fitBounds(results[0].geometry.viewport);

        const markerClickHandler = (marker) => {
          this.map.setZoom(15);
          this.map.setCenter(marker.getPosition());
        }


        this.markers = this.locations.map((location) => {
          const marker = new google.maps.Marker({
            ...location,
            map: this.map,
            icon: require('@/assets/marker.png')
          });
          marker.addListener('click', () => markerClickHandler(marker));

          return marker
        })
        new MarkerClusterer(this.map, this.markers, {imagePath: 'https://developers.google.com/maps/documentation/javascript/examples/markerclusterer/m',});
      })
    },
    setCurrentLocation () {
      this.centerOnLocation({ location: this.currentLocation })
    },
    getVisiblePoints () {
      var bounds = this.map.getBounds()
      this.export_items = []

      for (var i = 0; i < this.markers.length; i++) {
        var marker = this.markers[i]

        if (bounds.contains(marker.getPosition()) == true) {
          this.export_items.push({
            lat: marker.position.lat(),
            lng: marker.position.lng()
          })
        }
      }
    }
  },
  async mounted () {
    try {
      const google = await Maps()
      this.map = new google.maps.Map(document.getElementById('map'), {
        styles: MapTheme,
        zoomControl: true,
        mapTypeControl: false,
        scaleControl: false,
        streetViewControl: false,
        rotateControl: false,
        fullscreenControl: false
      });

      google.maps.event.addListener(this.map, 'idle', () => {
        // this.switchHeatmapMarkers(false)
      })

      var input = document.getElementById('search_box')
      var searchBox = new google.maps.places.SearchBox(input)
      searchBox.addListener('places_changed', () => {
        var results = searchBox.getPlaces()

        if (!results[0]) {
          throw new Error(status);
        }

        this.map.setCenter(results[0].geometry.location);
        this.map.fitBounds(results[0].geometry.viewport);
      })

      this.heatmap = new google.maps.visualization.HeatmapLayer({
        data: this.locations.map(l => new google.maps.LatLng(l.position.lat, l.position.lng)),
        gradient: gradient,
        radius: 15
      })

      this.centerOnLocation({ address: 'Miguel Hidalgo, Mexico City' })

      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(position => {
          try {
            this.currentLocation = new google.maps.LatLng(position.coords.latitude, position.coords.longitude)
            // this.setCurrentLocation()
          } catch (e) {
            console.error(e.message)
          }
        }, function (err) {
          console.error('Error getting current pos.')
        }, {
          enableHighAccuracy: true,
          timeout: 5000
        })
      }
    } catch (error) {
      console.error(error);
    }
  }
}
</script>

<style lang="scss" scoped>
  // #3498DB
  #map-container {
    width: 100%;
    height: 100%;
  }

  #map {
    width: 100%;
    height: 100%;
  }

  .app-toolbar {
    position: fixed;
    top: 2rem;
    left: 2rem;
    right: 2rem;
    z-index: 1000;
  }
</style>
