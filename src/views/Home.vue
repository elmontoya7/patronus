<template lang="pug">
  #map-container
    .app-toolbar
      b-row.align-items-center.justify-content-between
        b-col(cols="auto")
          b-card.px-3.py-2.mb-0(no-body)
            .d-flex.align-items-center
              v-icon.pointer.mr-3(name="ellipsis-v", scale="1.5", @click.native="filters_modal = true")
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
          b-button(variant="success", :disabled="!export_items.length", @click="exportCSV") Exportar a csv
        b-table(:items="export_items", outlined, show-empty, :fields="['title', 'description', 'category_name']")
          template(slot="empty")
            p.mb-0.text-secondary No hay registros en el área visible del mapa
    b-modal(size="lg", title="Crear nuevo registro", v-model="marker_modal", ok-title="Crear reporte", ok-variant="success", cancel-title="Cancelar", @ok="createReport")
      b-container(fluid)
        b-form-group(label="Nombre")
          b-form-input(placeholder="Nombre del registro", v-model="new_marker.title")
        b-form-group(label="Descripción")
          b-form-input(placeholder="Nombre del registro", v-model="new_marker.description")
        b-form-select(v-model="new_marker.category_name", :options="categories")
          template(slot="first")
            option(value="") -- Selecciona una categoría --
    b-modal(title="Filtrar reportes", ok-title="Aplicar", v-model="filters_modal")
      b-table(selectable, select-mode="single", :items="categories", :fields="['icon', 'text']")
        template(slot="icon", slot-scope="{ item }")
          v-icon(:name="item.icon")
        template(slot="text", slot-scope="{ item }")
          p.mb-0 {{ item.text }}
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

function convertToCSV(objArray) {
    var array = typeof objArray != 'object' ? JSON.parse(objArray) : objArray;
    var str = '';

    for (var i = 0; i < array.length; i++) {
        var line = '';
        for (var index in array[i]) {
            if (line != '') line += ','

            line += array[i][index];
        }

        str += line + '\r\n';
    }

    return str;
}

function exportCSVFile(headers, items, fileTitle) {
    if (headers) {
        items.unshift(headers);
    }

    // Convert Object to JSON
    var jsonObject = JSON.stringify(items);

    var csv = convertToCSV(jsonObject);

    var exportedFilenmae = fileTitle + '.csv' || 'export.csv';

    var blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    if (navigator.msSaveBlob) { // IE 10+
        navigator.msSaveBlob(blob, exportedFilenmae);
    } else {
        var link = document.createElement("a");
        if (link.download !== undefined) { // feature detection
            // Browsers that support HTML5 download attribute
            var url = URL.createObjectURL(blob);
            link.setAttribute("href", url);
            link.setAttribute("download", exportedFilenmae);
            link.style.visibility = 'hidden';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }
    }
}

export default {
  data () {
    return {
      export_modal: false,
      show_heat_map: false,
      map: null,
      heatmap: null,
      marker_modal: false,
      filters_modal: false,
      categories: [
        {
          text: 'Violencia',
          value: 'violence',
          icon: 'user-ninja'
        },
        {
          text: 'Vialidad afectada',
          value: 'vialidad afectada',
          icon: 'car-crash'
        },
        {
          text: 'Zona peligrosa',
          value: 'zona de peligro',
          icon: 'skull-crossbones'
        },
        {
          text: 'Ayuda',
          value: 'ayuda',
          icon: 'hospital-symbol'
        },
        {
          text: 'Transporte público',
          value: 'transporte público',
          icon: 'bus'
        }
      ],
      new_marker: {
        title: '',
        description: '',
        location: null,
        category_name: ''
      },
      locations: [
        {
          position: {
            lat: 19.451850,
            lng: -99.180264,
          },
          category_name: 'violence',
          title: 'Asalto con violencia',
          description: 'Dos personas armadas me quitaron mi cartera y celular'
        },
        {
          position: {
            lat: 19.447677,
            lng: -99.179425,
          },
          category_name: 'violence',
          title: 'Asalto con violencia en transporte',
          description: 'Asalto en transporte público en hora pico'
        },
        {
          position: {
            lat: 19.449650,
            lng: -99.196296,
          },
          category_name: 'violence',
          title: 'Asalto con violencia en vía pública',
          description: 'Una persona con arma de fuego me quito mi mochila con mi laptop'
        },
        {
          position: {
            lat: 19.447023,
            lng: -99.194167,
          },
          category_name: 'violence',
          title: 'Asalto con violencia',
          description: 'Dos personas me golperaron después de asaltarme'
        },
        {
          position: {
            lat: 19.445810,
            lng: -99.200618,
          },
          category_name: 'violence',
          title: 'Robo con violencia en vía pública',
          description: '3 personas con armas blancas robaron mis pertenencias'
        },
        {
          position: {
            lat: 19.445810,
            lng: -99.200618,
          },
          category_name: 'violence',
          title: 'Robo con violencia en vía pública',
          description: '3 personas con armas blancas robaron mis pertenencias'
        },
        {
          position: {
            lat: 19.447239,
            lng: -99.206881,
          },
          category_name: 'vialidad afectada',
          title: 'Transito lento',
          description: 'Transito lento por cruce de avenidas'
        },
        {
          position: {
            lat: 19.425948,
            lng: -99.192133,
          },
          category_name: 'vialidad afectada',
          title: 'Transito lento',
          description: 'Transito detenido por evento'
        },
        {
          position: {
            lat: 19.434522,
            lng: -99.201679,
          },
          category_name: 'vialidad afectada',
          title: 'Transito detenido',
          description: 'Mucho tráfico en hora pico, no se puede avanzar'
        },
        {
          position: {
            lat: 19.437665,
            lng: -99.183747,
          },
          category_name: 'vialidad afectada',
          title: 'Transito detenido por obras',
          description: 'Vialidad cerrada por obras en avenida'
        },
        {
          position: {
            lat: 19.440476,
            lng: -99.207083,
          },
          category_name: 'vialidad afectada',
          title: 'Transito lento',
          description: 'Demasiado tráfico durante la mañana'
        },
        {
          position: {
            lat: 19.440476,
            lng: -99.207083,
          },
          category_name: 'vialidad afectada',
          title: 'Transito lento',
          description: 'Demasiado tráfico durante la mañana'
        },
        {
          position: {
            lat: 19.427325,
            lng: -99.202450,
          },
          category_name: 'zona de peligro',
          title: 'Bache en el camino',
          description: 'Bache gigante sobre periferico'
        },
        {
          position: {
            lat: 19.424793,
            lng: -99.174731,
          },
          category_name: 'zona de peligro',
          title: 'Precaución, obras',
          description: 'Cuidado, obras en la zona'
        },
        {
          position: {
            lat: 19.424871,
            lng: -99.171261,
          },
          category_name: 'zona de peligro',
          title: 'Obras con maquinaria',
          description: 'Precación, obras con maquinaria pesada'
        },
        {
          position: {
            lat: 19.448856,
            lng: -99.177644,
          },
          category_name: 'zona de peligro',
          title: 'Zona de inseguridad',
          description: 'Delincuencia en la zona'
        },
        {
          position: {
            lat: 19.436467,
            lng: -99.169900,
          },
          category_name: 'zona de peligro',
          title: 'Zona peligrosa para ciclistas',
          description: 'Cruce difícil para ciclistas'
        },
        {
          position: {
            lat: 19.423800,
            lng: -99.165647,
          },
          category_name: 'ayuda',
          title: 'Violencia por bares',
          description: 'Enfrentamiento entre jovenes'
        },
        {
          position: {
            lat: 19.428587,
            lng: -99.191641,
          },
          category_name: 'ayuda',
          title: 'Accidente motociclista',
          description: 'Accidente de motociclista y automóvil'
        },
        {
          position: {
            lat: 19.433510,
            lng: -99.208644,
          },
          category_name: 'ayuda',
          title: 'Choque grave',
          description: 'Accidente de automóvil'
        },
        {
          position: {
            lat: 19.437784,
            lng: -99.192946,
          },
          category_name: 'ayuda',
          title: 'Primeros auxilios',
          description: 'Persona en vía pública necesita primeros auxilios'
        },
        {
          position: {
            lat: 19.424244,
            lng: -99.182251,
          },
          category_name: 'ayuda',
          title: 'Ciclista lesionado',
          description: 'Ciclista necesita ayuda por caida'
        },
        {
          position: {
            lat: 19.446608,
            lng: -99.191658,
          },
          category_name: 'transporte público',
          title: 'Choque en transporte público',
          description: 'Autobús de pasajeros tuve accidente víal'
        },
        {
          position: {
            lat: 19.433455,
            lng: -99.189770,
          },
          category_name: 'transporte público',
          title: 'Metro detenido',
          description: 'No avanza metro por problemas mecánicos'
        },
        {
          position: {
            lat: 19.436126,
            lng: -99.169813,
          },
          category_name: 'transporte público',
          title: 'Mal servicio de transporte',
          description: 'Pésimo servicio por parte de los conductores'
        },
        {
          position: {
            lat: 19.433212,
            lng: -99.184233,
          },
          category_name: 'transporte público',
          title: 'Accidente en unidad de transporte público',
          description: 'Leve choque entre unidad y automóvil'
        },
        {
          position: {
            lat: 19.422082,
            lng: -99.177968,
          },
          category_name: 'transporte público',
          title: 'Unidades de transporte muy lentas',
          description: 'Los choferes se quedan platicando'
        },
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
    exportCSV () {
      exportCSVFile({
        title: 'Titulo',
        description: 'Descripcion',
        category_name: 'Categoria',
        lat: 'Latitud',
        lng: 'Longitud'
      }, this.export_items.map(l => {
        return {
          title: l.title.replace(/,/gmi, ''),
          description: l.description.replace(/,/gmi, ''),
          category_name: l.category_name.replace(/,/gmi, ''),
          lat: l.lat,
          lng: l.lng
        }
      }), 'reportes')
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
            title: marker.title,
            description: marker.description,
            category_name: marker.category_name,
            lat: marker.position.lat(),
            lng: marker.position.lng()
          })
        }
      }
    },
    appendMarker (location) {
      new google.maps.Marker({
        position: location,
        map: this.map,
        icon: require('@/assets/marker.png')
      })
    },
    createReport () {
      this.locations.push({
        position: this.new_marker.location,
        category_name: this.new_marker.category_name,
        title: this.new_marker.title,
        description: this.new_marker.description
      })

      this.appendMarker(this.new_marker.location)

      this.new_marker = {
        location: null,
        title: '',
        description: '',
        category_name: ''
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

      google.maps.event.addListener(this.map, 'click', event => {
        this.marker_modal = true
        this.new_marker.location = event.latLng
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
        radius: 20
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
