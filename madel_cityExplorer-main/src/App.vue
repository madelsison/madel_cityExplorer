<script setup>
import { ref, watch, computed, nextTick } from 'vue';
import { 
  MapPin, Search, Loader2, AlertCircle, Activity, 
  Home, Info, Sun, Cloud, CloudRain, CloudSnow, 
  CloudDrizzle, AlertTriangle, CheckCircle, Camera, BookOpen
} from 'lucide-vue-next';

// --- Data ---
const query = ref('');
const loading = ref(false);
const error = ref(null);
const cityData = ref(null);
const suggestions = ref([]);
const showSuggestions = ref(false);
let debounceTimeout = null;
let abortController = null;

const fallbackImages = [
  'https://images.unsplash.com/photo-1477959858617-67f85cf4f1df?auto=format&fit=crop&w=1200&q=80',
  'https://images.unsplash.com/photo-1480714378408-67cf0d13bc1b?auto=format&fit=crop&w=1200&q=80',
  'https://images.unsplash.com/photo-1449824913935-59a10b8d2000?auto=format&fit=crop&w=1200&q=80'
];

// --- Methods ---

const getDayName = (dateString) => {
  return new Date(dateString).toLocaleDateString('en-US', { weekday: 'short' });
};

const getWeatherIconComponent = (code) => {
  if (code <= 1) return Sun;
  if (code <= 3) return Cloud;
  if (code <= 67) return CloudRain;
  if (code <= 77) return CloudSnow;
  return CloudDrizzle;
};

const getScoreColor = (score) => {
  if (score > 8) return 'bg-emerald-500';
  if (score > 6) return 'bg-teal-400';
  if (score > 4) return 'bg-yellow-400';
  return 'bg-orange-400';
};

const handleImageError = (e) => {
  const randomImg = fallbackImages[Math.floor(Math.random() * fallbackImages.length)];
  if (e.target.src !== randomImg) {
    e.target.src = randomImg;
  }
};

// Sort categories if available
const sortedCategories = computed(() => {
  if (!cityData.value || !cityData.value.categories) return [];
  return [...cityData.value.categories].sort((a, b) => b.score_out_of_10 - a.score_out_of_10);
});

// Robust API Fetcher for Teleport
const fetchWithFallback = async (url, signal) => {
  try {
    const res = await fetch(url, { signal });
    if (res.ok) return await res.json();
  } catch (err) { /* fail silently */ }

  try {
    const proxyUrl = `https://corsproxy.io/?${encodeURIComponent(url)}`;
    const res = await fetch(proxyUrl, { signal });
    if (res.ok) return await res.json();
  } catch (err) { /* fail silently */ }

  throw new Error("API Unavailable");
};

// --- NEW: Fetch City Image from Wikipedia (Smart Fallback) ---
const fetchWikiImage = async (searchQuery) => {
  try {
    // 1. Search for the page to get the correct title
    const searchRes = await fetch(`https://en.wikipedia.org/w/api.php?action=query&list=search&srsearch=${encodeURIComponent(searchQuery)}&format=json&origin=*`);
    const searchData = await searchRes.json();
    
    if (!searchData.query?.search?.length) return null;
    
    const title = searchData.query.search[0].title;
    
    // 2. Get the image for that page
    const imageRes = await fetch(`https://en.wikipedia.org/w/api.php?action=query&titles=${encodeURIComponent(title)}&prop=pageimages&format=json&pithumbsize=1000&origin=*`);
    const imageData = await imageRes.json();
    
    const pages = imageData.query.pages;
    const pageId = Object.keys(pages)[0];
    
    if (pages[pageId].thumbnail) {
      return pages[pageId].thumbnail.source;
    }
    return null;
  } catch (err) {
    return null;
  }
};

const fetchBackupPlaces = async (lat, lon) => {
  try {
    const url = `https://en.wikipedia.org/w/api.php?action=query&list=geosearch&gscoord=${lat}|${lon}&gsradius=10000&gslimit=10&format=json&origin=*`;
    const res = await fetch(url);
    const data = await res.json();
    
    if (data.query && data.query.geosearch) {
      return data.query.geosearch.map(place => ({
        name: place.title,
        type: 'Landmark', 
        dist: place.dist
      })).slice(0, 5);
    }
    return [];
  } catch (err) {
    return [];
  }
};

const fetchCityDetails = async (cityObj) => {
  loading.value = true;
  error.value = null;
  showSuggestions.value = false;
  cityData.value = null;
  let isBasicMode = false;

  try {
    const name = cityObj.name;
    const country = cityObj.country || 'Unknown';
    const population = cityObj.population;
    const lat = cityObj.latitude;
    const lon = cityObj.longitude;

    // 1. Weather (Reliable)
    const weatherUrl = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,weather_code,wind_speed_10m&daily=weather_code,temperature_2m_max,temperature_2m_min&temperature_unit=fahrenheit&wind_speed_unit=kmh&timezone=auto`;
    const weatherResponse = await fetch(weatherUrl);
    const weatherData = await weatherResponse.json();

    // 2. Urban Data (Teleport)
    let urbanData = { scores: { categories: [], teleport_city_score: 0 }, images: null, summary: null };
    let popularPlaces = []; 
    
    try {
      const locationsUrl = `https://api.teleport.org/api/locations/${lat},${lon}/`;
      const locationData = await fetchWithFallback(locationsUrl);
      const urbanAreaLink = locationData._embedded?.['location:nearest-urban-areas']?.[0]?._links['location:nearest-urban-area']?.href;

      if (urbanAreaLink) {
        const [scores, images, summary] = await Promise.all([
          fetchWithFallback(`${urbanAreaLink}scores/`),
          fetchWithFallback(`${urbanAreaLink}images/`),
          fetchWithFallback(urbanAreaLink)
        ]);
        urbanData = { scores, images, summary: summary.summary };
      } else {
        isBasicMode = true;
      }
    } catch (teleportErr) {
      isBasicMode = true;
    }

    // 3. IMAGE LOGIC: Try Teleport -> Then Wikipedia -> Then Fallback
    let finalImage = urbanData.images?.photos?.[0]?.image?.web;
    
    if (!finalImage) {
        // Try to find a Wikipedia image for "Cityname Country"
        const wikiImage = await fetchWikiImage(`${name} ${country}`);
        finalImage = wikiImage || fallbackImages[0];
    }

    // 4. PLACES LOGIC
    if (isBasicMode || !urbanData.scores.categories.length) {
      popularPlaces = await fetchBackupPlaces(lat, lon);
    }

    cityData.value = {
      name,
      country,
      population,
      summary: urbanData.summary || `Live weather reports and local insights for <strong>${name}</strong>. Data provided by Open-Meteo.`,
      totalScore: urbanData.scores.teleport_city_score || 0,
      image: finalImage,
      categories: urbanData.scores.categories || [],
      weather: weatherData,
      isBasicMode: isBasicMode,
      popularPlaces: popularPlaces 
    };

  } catch (err) {
    console.error(err);
    error.value = "Failed to load city data. Please check your connection.";
  } finally {
    loading.value = false;
  }
};

const handleSearchSubmit = () => {
  if (suggestions.value.length > 0) {
    fetchCityDetails(suggestions.value[0]);
  } else {
    const performSearch = async () => {
      try {
        const data = await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(query.value)}&count=1&language=en&format=json`).then(r => r.json());
        if (data.results && data.results.length > 0) {
          fetchCityDetails(data.results[0]);
        } else {
          error.value = "City not found.";
        }
      } catch (e) { error.value = "Search failed."; }
    };
    performSearch();
  }
};

const selectCity = (city) => {
  query.value = city.name;
  fetchCityDetails(city);
};

watch(query, (newQuery) => {
  if (abortController) abortController.abort();
  abortController = new AbortController();
  
  if (newQuery.length < 3) {
    suggestions.value = [];
    return;
  }

  clearTimeout(debounceTimeout);
  debounceTimeout = setTimeout(async () => {
    showSuggestions.value = true;
    try {
      const res = await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(newQuery)}&count=5&language=en&format=json`, { signal: abortController.signal });
      const data = await res.json();
      if (data.results) {
        suggestions.value = data.results;
      } else {
        suggestions.value = [];
      }
    } catch (err) { /* ignore aborts */ }
  }, 300);
});
</script>

<template>
  <div class="min-h-screen pb-12">
    <!-- Header -->
    <header class="bg-indigo-600 text-white py-8 px-4 shadow-lg rounded-b-[2.5rem] relative z-10">
      <div class="max-w-4xl mx-auto text-center">
        <h1 class="text-3xl md:text-4xl font-bold mb-2 flex items-center justify-center gap-2">
          <MapPin class="w-8 h-8" /> City Explorer
        </h1>
        <p class="text-indigo-100 mb-6">Live weather, quality of life scores, and global insights.</p>
        
        <div class="relative max-w-xl mx-auto text-slate-900">
          <form @submit.prevent="handleSearchSubmit" class="relative">
            <input
              type="text"
              v-model="query"
              placeholder="Search city (e.g. Baguio, New York)..."
              class="w-full pl-12 pr-4 py-4 rounded-full shadow-xl border-none focus:ring-4 focus:ring-indigo-400 outline-none text-slate-800 placeholder-slate-400"
            />
            <div class="absolute left-4 top-1/2 transform -translate-y-1/2 text-indigo-500">
              <Search class="w-6 h-6" />
            </div>
            <button type="submit" class="absolute right-2 top-2 bottom-2 bg-indigo-500 hover:bg-indigo-700 text-white px-6 rounded-full font-medium transition-colors">
              Search
            </button>
          </form>

          <!-- Suggestions -->
          <div v-if="showSuggestions && suggestions.length > 0" class="absolute w-full bg-white mt-2 rounded-xl shadow-2xl overflow-hidden z-50 border border-slate-100 text-left">
            <div
              v-for="city in suggestions"
              :key="city.id"
              @click="selectCity(city)"
              class="px-6 py-3 hover:bg-indigo-50 cursor-pointer border-b border-slate-50 last:border-0 text-slate-700 transition-colors"
            >
              <span class="font-bold">{{ city.name }}</span>, {{ city.country }}
              <span v-if="city.admin1" class="text-xs text-slate-400 ml-1">({{ city.admin1 }})</span>
            </div>
          </div>
        </div>
      </div>
    </header>

    <main class="max-w-5xl mx-auto px-4 mt-8">
      
      <!-- Loading -->
      <div v-if="loading" class="flex flex-col items-center justify-center py-20">
        <Loader2 class="w-12 h-12 text-indigo-500 animate-spin mb-4" />
        <p class="text-slate-500 animate-pulse">Fetching live data...</p>
      </div>

      <!-- Error -->
      <div v-else-if="error" class="bg-red-50 border-l-4 border-red-500 p-6 rounded-r-lg max-w-2xl mx-auto flex items-start gap-4">
        <AlertCircle class="w-6 h-6 text-red-500 mt-1" />
        <div>
          <h3 class="font-bold text-red-700">Error</h3>
          <p class="text-red-600">{{ error }}</p>
        </div>
      </div>

      <!-- Empty State -->
      <div v-else-if="!cityData" class="text-center py-20 text-slate-400">
        <div class="bg-white p-8 rounded-full w-32 h-32 mx-auto mb-6 flex items-center justify-center shadow-sm">
          <Activity class="w-16 h-16 text-indigo-100" />
        </div>
        <h2 class="text-2xl font-semibold text-slate-600">Ready to Explore?</h2>
        <p>Search for a city to see live weather and statistics.</p>
      </div>

      <!-- Content -->
      <div v-else class="animate-fade-in-up">
        
        <!-- Status Bar -->
        <div class="flex justify-end mb-2">
          <div v-if="cityData.isBasicMode" class="inline-flex items-center bg-amber-100 text-amber-800 text-xs font-bold px-3 py-1 rounded-full">
            <AlertTriangle class="w-3 h-3 mr-1" />
            Urban Data Unavailable - Basic Mode
          </div>
          <div v-else class="inline-flex items-center bg-green-100 text-green-800 text-xs font-bold px-3 py-1 rounded-full">
            <CheckCircle class="w-3 h-3 mr-1" />
            All Systems Online
          </div>
        </div>

        <!-- Hero Image -->
        <div class="relative h-64 md:h-80 rounded-[2rem] overflow-hidden shadow-xl mb-8 group bg-slate-200">
          <img 
            :src="cityData.image" 
            :alt="cityData.name" 
            class="w-full h-full object-cover transition-transform duration-700 group-hover:scale-105"
            @error="handleImageError"
          />
          <div class="absolute inset-0 bg-gradient-to-t from-slate-900/90 via-slate-900/30 to-transparent flex flex-col justify-end p-8">
            <h2 class="text-4xl md:text-6xl font-bold text-white mb-2">{{ cityData.name }}</h2>
            <div class="flex items-center text-indigo-200 text-lg font-medium">
              <MapPin class="w-5 h-5 mr-2" />
              {{ cityData.country }}
            </div>
          </div>
        </div>

        <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
          <!-- Left Column -->
          <div class="lg:col-span-2">
            
            <!-- Weather Card -->
            <div class="bg-white p-6 rounded-3xl shadow-sm border border-slate-100 mb-8">
              <div class="flex items-center justify-between mb-6">
                <div>
                  <p class="text-slate-500 font-medium">Current Weather</p>
                  <div class="flex items-center gap-3 mt-1">
                    <h3 class="text-4xl font-bold text-slate-800">{{ Math.round(cityData.weather.current.temperature_2m) }}°F</h3>
                    <component :is="getWeatherIconComponent(cityData.weather.current.weather_code)" class="w-8 h-8 text-yellow-500" />
                  </div>
                </div>
                <div class="text-right hidden sm:block">
                  <p class="text-sm text-slate-400">Wind Speed</p>
                  <p class="font-medium text-slate-600">{{ cityData.weather.current.wind_speed_10m }} km/h</p>
                </div>
              </div>

              <!-- Forecast -->
              <div class="grid grid-cols-5 gap-2 text-center divide-x divide-slate-100">
                <div v-for="(date, index) in cityData.weather.daily.time.slice(0, 5)" :key="date" class="flex flex-col items-center p-2">
                  <p class="text-xs text-slate-400 mb-2">{{ getDayName(date) }}</p>
                  <div class="mb-2 scale-75">
                    <component :is="getWeatherIconComponent(cityData.weather.daily.weather_code[index])" class="w-6 h-6 text-slate-400" />
                  </div>
                  <p class="text-sm font-semibold text-slate-700">{{ Math.round(cityData.weather.daily.temperature_2m_max[index]) }}°</p>
                  <p class="text-xs text-slate-400">{{ Math.round(cityData.weather.daily.temperature_2m_min[index]) }}°</p>
                </div>
              </div>
            </div>

            <!-- Stats -->
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-8">
              <div v-if="cityData.totalScore > 0" class="bg-white p-4 rounded-xl shadow-sm border border-slate-100 flex items-center space-x-4 hover:shadow-md transition-shadow">
                <div class="p-3 rounded-full bg-indigo-500 bg-opacity-10">
                  <Activity class="w-6 h-6 text-indigo-500" />
                </div>
                <div>
                  <p class="text-xs text-slate-500 uppercase font-bold tracking-wide">Urban Score</p>
                  <p class="text-slate-800 font-semibold">{{ Math.round(cityData.totalScore) }} / 100</p>
                </div>
              </div>
              <div class="bg-white p-4 rounded-xl shadow-sm border border-slate-100 flex items-center space-x-4 hover:shadow-md transition-shadow">
                <div class="p-3 rounded-full bg-emerald-500 bg-opacity-10">
                  <Home class="w-6 h-6 text-emerald-500" />
                </div>
                <div>
                  <p class="text-xs text-slate-500 uppercase font-bold tracking-wide">Population</p>
                  <p class="text-slate-800 font-semibold">{{ cityData.population ? cityData.population.toLocaleString() : 'N/A' }}</p>
                </div>
              </div>
            </div>

            <!-- Summary -->
            <div class="bg-white p-8 rounded-3xl shadow-sm border border-slate-100 mb-8">
              <h3 class="text-xl font-bold text-slate-800 mb-4 flex items-center gap-2">
                <Info class="w-5 h-5 text-indigo-500" /> About
              </h3>
              <div class="prose prose-slate max-w-none text-slate-600 leading-relaxed" v-html="cityData.summary"></div>
            </div>
            
            <!-- Categories -->
            <div v-if="cityData.categories.length > 0" class="bg-white p-8 rounded-3xl shadow-sm border border-slate-100">
              <h3 class="text-xl font-bold text-slate-800 mb-6">Life Quality Index</h3>
              <div class="grid grid-cols-1 md:grid-cols-2 gap-x-8 gap-y-4">
                <div v-for="cat in cityData.categories" :key="cat.name" class="mb-3">
                  <div class="flex justify-between mb-1">
                    <span class="text-sm font-medium text-slate-700">{{ cat.name }}</span>
                    <span class="text-sm font-medium text-slate-500">{{ cat.score_out_of_10.toFixed(1) }}/10</span>
                  </div>
                  <div class="w-full bg-slate-200 rounded-full h-2.5">
                    <div 
                      class="h-2.5 rounded-full transition-all duration-1000 ease-out" 
                      :class="getScoreColor(cat.score_out_of_10)"
                      :style="{ width: `${cat.score_out_of_10 * 10}%` }"
                    ></div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Right Column -->
          <div class="space-y-6">
            
            <!-- Popular Places (Logic: Show Teleport Highlights OR Backup Places) -->
            <div class="bg-indigo-900 text-white p-6 rounded-3xl shadow-lg">
              <h4 class="font-bold text-lg mb-4 border-b border-indigo-700 pb-2">
                City Highlights & Popular Places
              </h4>
              
              <!-- 1. Teleport Mode: Show High Scoring Categories -->
              <ul v-if="cityData.categories.length > 0" class="space-y-4">
                <li v-for="(cat, index) in sortedCategories.slice(0, 3)" :key="index" class="flex items-center gap-3">
                  <div class="w-8 h-8 rounded-full bg-indigo-700 flex items-center justify-center text-sm font-bold">
                    {{ index + 1 }}
                  </div>
                  <span class="font-medium">{{ cat.name }}</span>
                </li>
              </ul>

              <!-- 2. Basic Mode: Show Real Backup Places from Wikipedia -->
              <ul v-else-if="cityData.popularPlaces && cityData.popularPlaces.length > 0" class="space-y-4">
                 <li v-for="(place, index) in cityData.popularPlaces" :key="index" class="flex items-center gap-3">
                  <div class="w-8 h-8 rounded-full bg-indigo-700 flex-shrink-0 flex items-center justify-center text-sm font-bold">
                    <BookOpen class="w-4 h-4" />
                  </div>
                  <div class="flex flex-col">
                    <span class="font-medium text-sm">{{ place.name }}</span>
                    <span class="text-xs text-indigo-300 capitalize">Landmark / Notable Place</span>
                  </div>
                </li>
              </ul>

              <!-- 3. Empty State -->
              <div v-else class="text-indigo-200 text-sm">
                Explore the city center to find hidden gems and local spots.
              </div>
            </div>

            <div class="bg-white p-6 rounded-3xl shadow-sm border border-slate-100">
              <h4 class="font-bold text-slate-800 mb-4">Guide</h4>
              <div class="space-y-3 text-sm text-slate-600">
                <div class="flex items-center gap-2"><div class="w-3 h-3 rounded-full bg-emerald-500"></div><span>Excellent (8-10)</span></div>
                <div class="flex items-center gap-2"><div class="w-3 h-3 rounded-full bg-teal-400"></div><span>Good (6-8)</span></div>
                <div class="flex items-center gap-2"><div class="w-3 h-3 rounded-full bg-yellow-400"></div><span>Average (4-6)</span></div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>