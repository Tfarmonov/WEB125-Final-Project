<template>
  <div
  class="weather-container"
  :style="{ backgroundImage: `url(${backgroundImage})`, backgroundSize: 'cover', backgroundPosition: 'center' }"
>
    <div class="weather-wrap">
      <div class="unit-toggle">
        <label class="switch">
          <input type="checkbox" v-model="isFahrenheit" @change="toggleUnit" />
          <span class="slider round"></span>
        </label>
        <span class="unit-label">{{ isFahrenheit ? "°F" : "°C" }}</span>
      </div>
      <div class="search-box">
        <input
          type="text"
          placeholder="Search..."
          class="search-bar"
          v-model="query"
          v-on:keypress="fetchWeather"
        />
      </div>
      <div v-if="showSaveButton" class="save-location">
        <button @click="saveLocation">Save This Location</button>
      </div>
      <div class="saved-locations" v-if="savedLocations.length">
        <h4 style="color: white;">Saved Locations</h4>
        <ul>
          <li v-for="(loc, index) in savedLocations" :key="index">
            <div class="saved-item">
              <span @click="loadSavedLocation(loc)">{{ loc }}</span>
              <button class="delete-btn" @click.stop="removeSavedLocation(index)">✖</button>
            </div>
          </li>
        </ul>
      </div>
      <div class="view-toggle">
              <button @click="view = 'current'">Now</button>
              <button @click="getForecast('hourly')">Hourly</button>
              <button @click="getForecast('daily')">5-Day</button>
            </div>
        <!-- CURRENT WEATHER VIEW -->
        <div class="weather-info" v-if="view === 'current' && weather.main">
          <div class="location-box">
            <div class="location">
              {{ weather.name }}, {{ weather.state }}, {{ weather.sys.country }}
            </div>
            <div class="date">{{ todaysDate() }}</div>
          </div>
          <div class="weather-box">
            <div class="temp">{{ Math.round(weather.main.temp) }}{{ unitSymbols[unit] }}</div>
            <div class="weather">{{ weather.weather[0].main }}</div>
            <div class="icon">
              <img :src="`${weather_icon}${weather.weather[0].icon}@2x.png`" />
            </div>
          </div>
        </div>

        <!-- HOURLY FORECAST VIEW -->
        <div class="weather-info" v-if="view === 'hourly' && forecast.length">
          <div class="location-box">
            <div class="location">
              {{ weather.name }}, {{ weather.state }}, {{ weather.sys.country }}
            </div>
            <div class="date">{{ todaysDate() }}</div>
          </div>
          <h3 style="color: white; text-align: center;">Hourly Forecast</h3>

          <div class="chart-container">
            <canvas id="hourlyChart"></canvas>
          </div>
        </div>

        <!-- 5-DAY FORECAST VIEW -->
        <div class="weather-info" v-if="view === 'daily' &&   forecast.length">
          <div class="location-box">
            <div class="location">
              {{ weather.name }}, {{ weather.state }}, {{ weather.sys.country }}
            </div>
            <div class="date">{{ todaysDate() }}</div>
          </div>
          <h3 style="color: white; text-align: center;">5-Day Forecast</h3>

          <div class="daily-forecast">
            <div v-for="(day, index) in groupByDay(forecast)" :key="index" class="day-box">
              <div class="day-date">{{ day.date }}</div>
              <div class="day-icon">
                <img :src="`${weather_icon}${day.icon}@2x.png`" alt="Weather Icon" />
              </div>
              <div class="day-temps">
                <span>High: {{ day.high }}{{ unitSymbols[unit] }}</span>
                <span>Low: {{ day.low }}{{ unitSymbols[unit] }}</span>
              </div>
            </div>
          </div>
        </div>
  </div>
</div>
</template>

<script>
import axios from "axios";
import { Chart, registerables } from 'chart.js';
Chart.register(...registerables);

Chart.defaults.color = '#FFF'; 
export default {
  data() {
    return {
      api_key: "27bcc252742830381afd6856832da01a",
      url_base: "https://api.openweathermap.org/data/2.5/",
      weather_icon: "http://openweathermap.org/img/wn/",
      query: "",
      weather: {},
      unit: "metric",
      backgroundImage: 'https://www.storyboardthat.com/photo-download/get-photo?data=cGJfMjQwMTQ1OHxodHRwczovL3BpeGFiYXkuY29tL2dldC9nOWYwZDAzZDE3NDE1Yzc4ZTNkMWYxYTE0OGQxYmQ3OTljODBmYWY4OWU5Njc1M2I4Y2FiMDY2MDlkM2VkYTZhYjM5MTVkY2IwYWE5OTU1YTZhZmRjM2MwZDU4MGE3ZWJkZWM0NWUyNDk0ODdkN2VjYzQxOWQ0ZjAyODc0YzFiNDZfMTI4MC5qcGd8aHR0cHM6Ly9waXhhYmF5LmNvbS9pbGx1c3RyYXRpb25zL21hcC1vZi10aGUtd29ybGQtYmFja2dyb3VuZC1wYXBlci0yNDAxNDU4L3xZdXJpX0I%3D',
      unitSymbols: {
        metric: "°C",
        imperial: "°F",
      },
      view: "current",
      forecast: [],
      savedLocations: [],
      showSaveButton: false,
    };
  },
  mounted() {
    const saved = localStorage.getItem("savedLocations");
    if (saved) {
      this.savedLocations = JSON.parse(saved);
    }
  },
  computed: {
    isFahrenheit: {
      get() {
        return this.unit === "imperial";
      },
      set(value) {
        this.unit = value ? "imperial" : "metric";
      },
    },
  },
  methods: {
    setBackground(weather) {
      const main = weather.weather[0].main.toLowerCase();
      const temp = weather.main.temp;

      if (main.includes('rain')) {
        this.backgroundImage = "https://www.storyboardthat.com/photo-download/get-photo?data=cGJfMzIxNjYwN3xodHRwczovL3BpeGFiYXkuY29tL2dldC9nMjczOWQ5NTA1MTlmMjU5OTJjMWU3MjZiYjhjNzgxOGQ1MmQ4ZmFlM2RhMDlhY2ZkNDk1YmVmOTU4MzIyMzcwOGY0MjI5MDI1ZTQ3NjljM2ZlYzEwNWNmMGJjZTI0NzNiNWI0ZjFiMGMyNmZhNWY3ZTVkODdlZjE2YjljZTFiZmFfMTI4MC5qcGd8aHR0cHM6Ly9waXhhYmF5LmNvbS9waG90b3MvcmFpbmRyb3BzLXdpbmRvdy1yYWlueS1kYXktMzIxNjYwNy98Sm9zaHVhX3NlYWp3OTI%3D";
      } else if (temp <= 10) {
        this.backgroundImage = "https://www.storyboardthat.com/photo-download/get-photo?data=cGJfNjI3MjM2MnxodHRwczovL3BpeGFiYXkuY29tL2dldC9nNmMzOTk5Y2JiYzU0ZTY2ZDRmMGViOTY1YTM3Mjc4M2U1Mjk5YjhiYWU2MjYzNDJjYTU0MjNiNWZjMDQ2OWNhYzM4ZDk2MDQ3ZmIxNDdiZDA0YWVhMjljNGFjOTg4ZmQ0OGEyOGVmZmZlYmYyOTQ0OTdlMmY1NTQ1N2NiYmNjNzZfMTI4MC5qcGd8aHR0cHM6Ly9waXhhYmF5LmNvbS9waG90b3MvbW91bnRhaW5zLXNub3ctcGVhay1zdW1taXQtNjI3MjM2Mi98anV1dW5tYXQ%3D";
      } else {
        this.backgroundImage = "https://www.storyboardthat.com/photo-download/get-photo?data=cGJfMTc2ODk2N3xodHRwczovL3BpeGFiYXkuY29tL2dldC9nYzIzNTA3NWRmZTVmMzQ1ZDk1NmM3NWVlMGIwNmRlZTg0MWJmYzgwYzgwZjNjYWVmYTMyNjgyZDcyZGI4NWQyOGIzNGMyNjI5NzczNWI2NmQ1YTVhN2IzMGNmNWE0OWZkOWVhODJhMWJjZjY0ZjFiZGYzMTIwZTcxMDViMTI5NzhfMTI4MC5qcGd8aHR0cHM6Ly9waXhhYmF5LmNvbS9waG90b3MvY2xvdWRzLXNreS13ZWF0aGVyLXRodW5kZXJzdG9ybS0xNzY4OTY3L3xwaHRvcnhw";
      }
    },
    async fetchWeather(e) {
      if (e.key === "Enter") {
        const cleanedQuery = this.query.trim();
        try {

          const geoResponse = await axios.get(
            `https://api.openweathermap.org/geo/1.0/direct?q=${cleanedQuery}&limit=1&appid=${this.api_key}`
          );
          const locationData = geoResponse.data[0];
          const { name, state, country, lat, lon } = locationData;

          const weatherResponse = await axios.get(
            `${this.url_base}weather?lat=${lat}&lon=${lon}&units=${this.unit}&appid=${this.api_key}`
          );


          weatherResponse.data.name = name;
          weatherResponse.data.state = state;
          weatherResponse.data.sys.country = country;
          this.setResults(weatherResponse.data);
          this.showSaveButton = true;
        } catch (error) {
          console.error("Location not found:", error);
          alert("Location not found. Try format: City, State, Country.");
        }
      }
    },
    async loadSavedLocation(location) {
      this.query = location;
      this.fetchWeather({ key: "Enter" });
    },
    async removeSavedLocation(index) {
      this.savedLocations.splice(index, 1);
      localStorage.setItem("savedLocations", JSON.stringify(this.savedLocations));
    },
    formatTime(datetime) {
      return new Date(datetime).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    },
    groupByDay(list) {
      const grouped = {};

      list.forEach(item => {
        const date = item.dt_txt.split(" ")[0];
        if (!grouped[date]) grouped[date] = [];
        grouped[date].push(item);
      });

      return Object.keys(grouped).map(dateStr => {
        const dayData = grouped[dateStr];

        const temps = dayData.map(item => item.main.temp);
        const high = Math.max(...temps);
        const low = Math.min(...temps);


        const icon = dayData[0].weather[0].icon;

        const dateObj = new Date(dateStr);
        const options = { weekday: 'long', month: 'short', day: 'numeric' };
        const formattedDate = dateObj.toLocaleDateString(undefined, options);

        return {
          date: formattedDate,
          high: Math.round(high),
          low: Math.round(low),
          icon: icon
        };
      });
    },
    async getForecast(type) {
      if (!this.weather.coord) {
        alert("Search for a location first.");
        return;
      }

      const { lat, lon } = this.weather.coord;

      try {
        const response = await axios.get(
          `${this.url_base}forecast?lat=${lat}&lon=${lon}&units=${this.unit}&appid=${this.api_key}`
        );
        this.forecast = response.data.list;
        this.view = type;

        if (type === 'hourly') {

          setTimeout(this.createHourlyChart, 100);
        }
      } catch (error) {
        console.error("Forecast error:", error);
        alert("Could not fetch forecast.");
      }
    },
    saveLocation() {
      const fullName = `${this.weather.name}, ${this.weather.sys.country}`;

      if (!this.savedLocations.includes(fullName)) {
        this.savedLocations.push(fullName);
        localStorage.setItem("savedLocations", JSON.stringify(this.savedLocations));
      }

      this.showSaveButton = false; 
    },
    setResults(returnedResponse) {
      this.weather = returnedResponse;
      this.showSaveButton = true;
      this.setBackground(returnedResponse);
    },
    todaysDate() {
      const months = [
        "Jan", "Feb", "Mar", "Apr", "May", "Jun",
        "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"
      ];
      const days = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
      let d = new Date();
      let month = months[d.getMonth()];
      let day = days[d.getDay()];
      let date = d.getDate();
      let year = d.getFullYear();
      return `${month} ${date} ${day} ${year}`;
    },
    toggleUnit() {
      if (this.query) {
        this.fetchWeather({ key: "Enter" });
      }
    },
    createHourlyChart() {
      const ctx = document.getElementById('hourlyChart');

      const temps = this.forecast.slice(0, 8).map(item => item.main.temp);
      const rains = this.forecast.slice(0, 8).map(item => item.rain ? item.rain['3h'] || 0 : 0);
      const labels = this.forecast.slice(0, 8).map(item => this.formatTime(item.dt_txt));

      if (this.chartInstance) {
        this.chartInstance.destroy();
      }

      this.chartInstance = new Chart(ctx, {
        type: 'line',
        data: {
          labels: labels,
          datasets: [
            {
              label: 'Temperature',
              data: temps,
              backgroundColor: 'rgba(54, 162, 235, 0.5)',
              borderColor: 'rgba(54, 162, 235, 1)',
              borderWidth: 1,
              yAxisID: 'y',
            },
            {
              label: 'Rain (in)',
              data: rains,
              backgroundColor: 'rgba(0, 0, 0, 0.5)',
              borderColor: 'rgba(0, 0, 0, 1)',
              borderWidth: 1,
              yAxisID: 'y',
            }
          ]
        },
        options: {
          responsive: true,
          interaction: {
            mode: 'index',
            intersect: false,
          },
          scales: {
            y: {
              type: 'linear',
              position: 'left',
              title: { display: true, text: 'Temp (°)' },
            },
            y1: {
              type: 'linear',
              position: 'right',
              title: { display: true, text: 'Rain (in)' },
              grid: { drawOnChartArea: false },
            }
          }
        }
      });
    },
  },
};
</script>

<style>
@import url("https://fonts.googleapis.com/css2?family=Montserrat:wght@300;500;700;900&display=swap");
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: "Montserrat";
}
.weather-container {
  width: 100%;
  height: 100vh;
  background-size: cover;
  background-position: center;
  transition: 0.4s;
  box-shadow: 0px 0px 30px #00000065;
}
.weather-wrap {
  height: 100%;
  padding: 25px;
  border-radius: 25px;
  background-image: linear-gradient(
    to bottom,
    rgba(0, 0, 0, 0.15),
    rgba(0, 0, 0, 0.4)
  );
}
.search-box .search-bar {
  display: block;
  width: 100%;
  padding: 15px;
  color: #313131;
  font-size: 20px;
  appearance: none;
  border: none;
  outline: none;
  background: none;
  background-color: rgba(255, 255, 255, 0.5);
  box-shadow: 0px 0px 8px rgba(0, 0, 0, 0.25);
  border-radius: 10px;
  transition: 0.4s;
}
.search-box .search-bar:focus {
  box-shadow: 0px 0px 16px rgba(0, 0, 0, 0.25);
  background-color: rgba(255, 255, 255, 0.75);
}
.location-box .location {
  color: #fff;
  font-size: 32px;
  font-weight: 500;
  font-style: italic;
  text-align: center;
  margin-top: 30px;
}
.location-box .date {
  color: #fff;
  font-size: 20px;
  font-weight: 300;
  text-align: center;
}
.weather-box {
  text-align: center;
}
.weather-box .temp {
  display: inline-block;
  padding: 10px 25px;
  color: #fff;
  font-size: 102px;
  font-weight: 900;
  text-shadow: 3px 6px rgba(0, 0, 0, 0.25);
  background-color: rgba(255, 255, 255, 0.25);
  border-radius: 16px;
  margin: 30px 0px;
  box-shadow: 3px 6px rgba(0, 0, 0, 0.25);
  font-style: italic;
}
.weather-box .weather {
  color: #fff;
  font-size: 48px;
  font-weight: 700;
  font-style: italic;
  text-shadow: 3px 6px rgba(0, 0, 0, 0.25);
}
.forecast-box {
  margin: 8px 0;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
}

.daily-forecast {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 15px;
  margin-top: 20px;
}

.day-box {
  background-color: rgba(255, 255, 255, 0.15);
  border-radius: 12px;
  padding: 15px;
  height: auto;
  width: 250px;
  text-align: center;
  color: white;
}

.day-date {
  font-weight: bold;
  margin-bottom: 15px;
  font-size: 25px;
}

.day-temps span {
  display: block;
  margin: 20px 0;
  font-size: 20px;
  justify-content: center;
}

.day-icon img {
  width: 150px;
  height: 150px;
  margin: 5px 0;
}

.unit-toggle {
  text-align: center;
  margin: 15px 0;
}

.switch {
  position: relative;
  display: inline-block;
  width: 60px;
  height: 34px;
  vertical-align: middle;
}

.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #ccc;
  transition: 0.4s;
  border-radius: 34px;
}

.slider:before {
  position: absolute;
  content: "";
  height: 26px;
  width: 26px;
  left: 4px;
  bottom: 4px;
  background-color: white;
  transition: 0.4s;
  border-radius: 50%;
}

input:checked + .slider {
  background-color: #4caf50;
}

input:checked + .slider:before {
  transform: translateX(26px);
}

.unit-label {
  color: white;
  margin-left: 12px;
  font-size: 16px;
  font-weight: 500;
  vertical-align: middle;
}

.view-toggle {
  text-align: center;
  margin-top: auto;
}
.view-toggle button {
  background-color: rgba(0, 0, 0, 1);
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 20px;
  padding: 6px 12px;
  color: white;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.3s;
}
.view-toggle button:hover {
  background-color: rgba(0, 0, 0, 0.4);
}

.save-location {
  text-align: center;
  margin: 15px 0;
}

.save-location button {
  background-color: rgba(255, 255, 255, 0.2);
  border: none;
  padding: 10px 20px;
  border-radius: 20px;
  color: white;
  font-weight: bold;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.save-location button:hover {
  background-color: rgba(255, 255, 255, 0.4);
}

.saved-locations ul {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
  padding: 0;
  list-style: none;
  margin: 10px 0;
}

.saved-locations h4 {
  color: white;
  text-align: center;
}

.saved-item {
  display: flex;
  align-items: center;
  background-color: rgba(0, 0, 0, 1);
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 20px;
  padding: 6px 12px;
  color: white;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.3s;
}

.saved-item span {
  cursor: pointer;
}

.saved-item span:hover {
  text-decoration: underline;
}

.delete-btn {
  background: none;
  border: none;
  color: #ff6666;
  font-size: 14px;
  cursor: pointer;
  padding: 0;
  display: none;
}

.saved-item:hover .delete-btn {
  display: inline;
}
.chart-container {
  position: relative;
  width: 90%;
  max-width: 900px;
  height: 600px; 
  margin: 0 auto 20px;
}

#hourlyChart {
  width: 100%;
  height: 100%;
}
</style>