<template>
  <div>
    <div style="display: flex; justify-content: space-between; padding: 10px; background-color: #007bff; color: white;">
      <div>
        <input v-model="startCoordinate" placeholder="Start Coordinate (lng, lat)" />
        <input v-model="endCoordinate" placeholder="End Coordinate (lng, lat)" />
      </div>
      <button @click="drawRoute" style="background-color: green; color: white;">Get Route</button>
    </div>
    <div id="map" class="map"></div>
  </div>
</template>
<script>
import 'ol/ol.css';
import { fromLonLat } from 'ol/proj';
import { Map, View } from 'ol';
import TileLayer from 'ol/layer/Tile';
import XYZ from 'ol/source/XYZ';
import Feature from 'ol/Feature';
import Point from 'ol/geom/Point';
import VectorSource from 'ol/source/Vector';
import VectorLayer from 'ol/layer/Vector';
import Style from 'ol/style/Style';
import Icon from 'ol/style/Icon';
import axios from 'axios';
import { LineString } from 'ol/geom'; // Add this to draw the route
import Stroke from 'ol/style/Stroke';  

export default {
  name: 'MapComponent',
  data() {
    return {
      startCoordinate: '',
      endCoordinate: '',
      map: null, //openlayers harita nesnesini tutar.
      markerLayer: null, // harita üzerindeki markerın bulunduğu katmanı tutar.
      routeLayer: null, //harita üzerindeki rotaların bulunduğu katmanı tutar
      apiKey: '5b3ce3597851110001cf62487eef3868a5ca4248a7e779db7f0c4cd9'
    };
  },
  mounted() {
    this.initMap(); //haritayı başlatan method
  },
  methods: {
    initMap() {
      this.map = new Map({
        target: 'map', //haritanın hangi html elemanına yerleştirileceğini belirtir. 
        layers: [ //haritaya eklenen google maps katmanı
          new TileLayer({
            source: new XYZ({
              url: 'https://mt0.google.com/vt/lyrs=m&hl=$tr&scale=4&apistyle=s.t%3A2%7Cs.e%3Al%7Cp.v%3Aoff&x={x}&y={y}&z={z}', // Google Maps 
            }),
          }),
        ],
        view: new View({ //haritanın başlangıç merkezini ve zoom seviyesini ayarlar. 
          center: [3262491, 4865942],
          zoom: 6,
        }),
      });

      this.markerLayer = new VectorLayer({
        source: new VectorSource(),
        style: new Style({
          image: new Icon({
            anchor: [0.5, 1],
            src: require('@/assets/marker-icon.png'),
            scale: 0.1,
          }),
        }),
      });

      this.map.addLayer(this.markerLayer); //marker katmanı haritaya eklenir.

      this.routeLayer = new VectorLayer({
        source: new VectorSource(),
        style: new Style({
          stroke: new Stroke({
            color: '#FF0000',
            width: 3,
          }),
        }),
      });

      this.map.addLayer(this.routeLayer);

      // Click event to update coordinates
      this.map.on('click', this.handleMapClick);
    },
    handleMapClick(event) { //harita üzerinde tıklama olayı gerçekleştiğinde çağırılan bir fonksiyondur. 
      const clickedCoordinate = event.coordinate; //tıklanılan koordinatları depolar. 
      this.markerLayer.getSource().clear(); // Clear previous markers
      const marker = new Feature(new Point(clickedCoordinate)); // Point nesnesi harita üzerindeki belirli bir noktayı temsil eder.
      // Feature nesnesi open layersda harita üzerinde gösterilebilecek herhangi bir geometrik şekli temsil eder. Bu satır yeni bir point nesnesi eklendiği anlamına gelir. 
      this.markerLayer.getSource().addFeature(marker); //aritada tıklanan noktada bir işaretleyici eklemenizi sağlar. (iyice anla)

      axios.post('http://127.0.0.1:5000/save_coordinates', {
        coordinates: clickedCoordinate
      })
      .then(response => {
          console.log('Coordinates sent successfully: ', response.data);
      })
      .catch(error => {
          console.error('An error occurred while sending coordinates: ', error);
      });
    },
    drawRoute() {
    if (!this.startCoordinate || !this.endCoordinate) {
      alert('Please enter both start and end coordinates.');
      return;
    }

    console.log("Start Coordinate:", this.startCoordinate);
    console.log("End Coordinate:", this.endCoordinate);

    //kullanıcıdan alınan koordinat stringlerini alır, bunları ikiye böler (enlem ve boylam olarak) ve her bir değeri bir sayıya (number) dönüştürür.
    const start = this.startCoordinate.split(',').map(Number);
    const end = this.endCoordinate.split(',').map(Number);

    if (isNaN(start[0]) || isNaN(start[1]) || isNaN(end[0]) || isNaN(end[1])) {
      alert('Please enter valid coordinates.');
      return;
    }

    const apiUrl = `https://api.openrouteservice.org/v2/directions/driving-car?api_key=${this.apiKey}&start=${start.reverse().join(',')}&end=${end.reverse().join(',')}`;

    console.log("API URL: ", apiUrl);

    axios.get(apiUrl)
      .then(response => {
        console.log("API Response: ", response);

        //Bu satır, API yanıtının geçerli olup olmadığını kontrol eder:
        if (response.data && response.data.features && response.data.features.length > 0) {
          const routeCoordinates = response.data.features[0].geometry.coordinates; //geometry.coordinates, rota boyunca bulunan tüm noktaları içerir.
          const transformedCoordinates = routeCoordinates.map(coord => fromLonLat([coord[0], coord[1]])); //Bu fonksiyon, enlem ve boylamı OpenLayers'in desteklediği projeksiyon sistemine çevirir.

          const route = new LineString(transformedCoordinates); //LineString nesnesi rota boyunca çizilecek olan hattı temsil eder.
          this.routeLayer.getSource().clear();
          this.routeLayer.getSource().addFeature(new Feature({ geometry: route }));

          this.map.getView().fit(route.getExtent(), { duration: 1000 });

          axios.post('http://127.0.0.1:5000/save_route', {
            startCoordinate: this.startCoordinate,
            endCoordinate: this.endCoordinate
          })
          .then(response => {
              console.log('Route sent successfully: ', response.data);
          })
          .catch(error => {
              console.error('An error occurred while sending the route: ', error);
          });
        } else {
          console.error('No route found in the API response');
          alert('No route found. Please check the coordinates or try different ones.');
        }
      })
      .catch(error => {
        console.error('An error occurred while retrieving the route: ', error);
        alert('An error occurred while retrieving the route. Please try again.');
      });
  },
  },
};
</script>
  
  <style scoped>
  .map {
    width: 100%;
    height: 100vh; /* Haritayı tam ekran yapar */
  }
  </style>
  