<template>
  <div id="map" />
  <div id="nav">
    <button @click.prevent="handleStart"> {{ isStart ? "Стоп" :  "Пуск"}}</button>
    <div class="left-block">
      <span>Скорость: </span>
      <select @change="handleSelect" size="1"  v-model="selected">
        <option value="10">10</option>
        <option value="20">20</option>
        <option value="50">50</option>
        <option value="100">100</option>
      </select>
    </div>
  </div>
</template>

<script>
// <script lang='ts'>
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";
import { ACCESS_TOKEN } from './accessToken.ts';
import car from './assets/car.png';
import * as turf from '@turf/turf';
import axios from 'axios';

export default {
  data() {
    return {
      isStart: true,
      selected: 10,
      coordinates: [],
      lines: {
        "type": "Feature",
        "properties": {},
        "geometry": {
          "type": "LineString",
          "coordinates": this.coordinates
        }
      },
      next: 0,
      map: {},
      arc: [],
      steps: 1000,
      counter: 0,
      origin: [],
      destination: [],
      point: {},
      route: {},
      lineDistance: {},
    }
  },
  mounted() {
    mapboxgl.accessToken = ACCESS_TOKEN;

    const renderNav = () => {
      this.$data.map.addControl(new mapboxgl.NavigationControl({showCompass: true, howZoom: true }), 'bottom-right');
    }
    const getCoordinates = (arr) => {
      const arrNew = [];
      for (let i = 0; i < arr.length; i++) {
        if (i === 0) arrNew.push(arr[i])
        else if (arr[i][0] === arr[i-1][0] && arr[i][1] === arr[i-1][1]) continue;
        else arrNew.push(arr[i])
      }
      return arrNew;
    }
    const setStartEnd = (start, end) => {
      this.$data.origin = [...start];
      this.$data.destination = [...end];
    }
    const renderMap = () => {
      this.$data.map = new mapboxgl.Map({
          container: 'map',
          style: 'mapbox://styles/mapbox/streets-v11',
          center: this.$data.origin,
          zoom: 16
        })
    }
    const renderCar = (coordinates) => {
      setPoint(coordinates);

      this.$data.map.addSource('point', {
          'type': 'geojson',
          'data': this.$data.point
        });     
      this.$data.map.loadImage(car, (error, image) => {
        if (!error) this.$data.map.addImage("car", image);
      });
      this.$data.map.addLayer({
        'id': 'point',
        'source': 'point',
        'type': 'symbol',
        'layout': {
          'icon-image': 'car',
          'icon-rotate': ['get', 'bearing'],
          'icon-rotation-alignment': 'map',
          'icon-allow-overlap': true,
          'icon-ignore-placement': true,
          'icon-size': .05, 
        }
      });
    }
    const renderLines = (linesArr) => {
      this.$data.map.addSource('route2', {
        'type': 'geojson',
        'data': linesArr
      });
      this.$data.map.addLayer({
        "id": "route2",
        "type": "line",
        "source": "route2",
        "layout": {
          "line-join": "round",
          "line-cap": "round"
        },
        "paint": {
          "line-color": "green",
          "line-width": 3
        }
      })
    }
    const renderLine = (start, end, name) => {
      setRoute(start, end);
      
      this.$data.map.addSource(name + 1, {
        'type': 'geojson',
        'data': this.$data.route
      });
    }
    const setPoint = (coordinates) => {
      this.$data.point = {
        'type': 'FeatureCollection',
        'features': [
          {
            'type': 'Feature',
            'properties': {},
            'geometry': {
              'type': 'Point',
              'coordinates': coordinates
            }
          }
        ]
      };
    }
    const setRoute = (start, end) => {
      this.$data.route = {
        'type': 'FeatureCollection',
        'features': [
          {
            'type': 'Feature',
            'geometry': {
              'type': 'LineString',
              'coordinates': [start, end]
            }
          }
        ]
      };
      calcLineDistance(start, end);
      this.$data.route.features[0].geometry.coordinates = [...this.$data.arc];
      this.$data.route = {...this.$data.route}
    }
    const calcLineDistance = (start, end) => {
      this.$data.lineDistance = turf.length(turf.lineString([start, end]));
      const arrNew = [];

      for (let i = 0; i < this.$data.lineDistance; i += this.$data.lineDistance / this.$data.steps) {
        const segment = turf.along(this.$data.route.features[0], i);
        arrNew.push(segment.geometry.coordinates);
      }
      this.$data.arc = [...arrNew];
    }
    const animate = () =>  {
      
      const start =  this.$data.route.features[0].geometry.coordinates[
        this.$data.counter >= this.$data.steps ? this.$data.counter - 1 : this.$data.counter
      ];
      const end =  this.$data.route.features[0].geometry.coordinates[
        this.$data.counter >= this.$data.steps ? this.$data.counter : this.$data.counter + 1
      ];

      if (!start || !end) {
        this.$data.counter = 0;
        this.$data.next++;
        this.$data.origin = this.$data.lines.geometry.coordinates[this.$data.next];
        this.$data.destination = this.$data.lines.geometry.coordinates[this.$data.next + 1];

        renderLine(this.$data.origin, this.$data.destination, `route${this.$data.next}`);
        animate(this.$data.counter);
        return;
      }

      this.$data.point.features[0].geometry.coordinates =
        this.$data.route.features[0].geometry.coordinates[this.$data.counter];

      this.$data.point.features[0].properties.bearing = turf.bearing(
        turf.point(start),
        turf.point(end)
      );
      this.$data.map.flyTo({
        center: this.$data.point.features[0].geometry.coordinates,
        essential: true 
      });

      this.$data.point = {...this.$data.point};
      
      this.$data.map.getSource('point').setData(this.$data.point);
      
      if (this.$data.counter < this.$data.steps && this.$data.isStart) {
        requestAnimationFrame(animate);
      }
      
      this.$data.counter = this.$data.counter + 1;
    }
    const startRender = () => {
      this.$data.map.on('load', () => {
        renderNav();
        renderLines(this.$data.lines);
        renderLine(this.$data.origin, this.$data.destination, 'route');
        renderCar(this.$data.origin);
        setInterval(() => {
          if(this.$data.isStart) animate(this.$data.counter);
        }, 100);
        
      })
    }
    axios.get('https://bibix.gitlab.io/cdu-test/coordinates.ts')
      // .then(res => res.blob())
      // .then(res => res.text())
      .then(res => res.data )
      .then(res => this.$data.coordinates = [...getCoordinates(JSON.parse(res.slice(res.indexOf('[['))).map(el => [el[1], el[2]]))])
      .then(() => {
        this.$data.lines = {...this.$data.lines, geometry: {...this.$data.lines.geometry, coordinates: this.$data.coordinates}};
        setStartEnd(this.$data.coordinates[0], this.$data.coordinates[1]);
        renderMap();
        startRender();
      })
      .then(() => {
        const str = '';
        // str = this.$data.coordinates.reduce((red, el) => red = red + [el] + ';' , '');
        fetch(`https://api.mapbox.com/directions/v5/mapbox/cycling/${str}?geometries=geojson&access_token=${ACCESS_TOKEN}`)
          .then(res => res.json())
          .then(res => console.log(res))
      })
      .catch(err => console.log(err))
  },
  methods: {
    handleStart() {
      this.isStart = !this.isStart;
    },
    handleSelect(e) {
      this.$data.selected = e.target.value;
      this.$data.steps = 10000 / this.$data.selected
    }
  },
  
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
}
#map {
  height: 100vh;
  position: relative;
}
#nav {
  position: absolute;
  display: flex;
  justify-content: space-between;
  min-width: 300px;
  top: 0;
  border-radius: 10px;
  margin: 10px;
  padding: 15px;
  background: white;
  border: 1px solid black;
}
#nav button {
  padding: 5px 20px;
}
.left-block {
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>