<script>
const API_KEY = import.meta.env.VITE_OWM_API_KEY || ''
const GEO_URL = 'https://api.openweathermap.org/geo/1.0/direct'
const CURRENT_URL = 'https://api.openweathermap.org/data/2.5/weather'
const FORECAST_URL = 'https://api.openweathermap.org/data/2.5/forecast'
const HISTORY_KEY = 'clair-meteo-history-v1'

export default {
  name: 'WeatherApp',
  data: () => ({ recherche: '', suggestions: [], suggestionError: '', suggestionsLoading: false, ville: null, current: null, forecast: [], loading: false, error: '', units: 'metric', history: [], debounceTimer: null, searchController: null, requestId: 0 }),
  computed: {
    unitLabel() { return this.units === 'metric' ? '°C' : '°F' },
    windLabel() { return this.units === 'metric' ? 'm/s' : 'mph' },
    groupedForecast() {
      const groups = new Map()
      for (const item of this.forecast) { const day = item.dt_txt.slice(0, 10); if (!groups.has(day)) groups.set(day, []); groups.get(day).push(item) }
      return [...groups.entries()].slice(0, 5).map(([date, entries]) => {
        const noon = entries.find((item) => item.dt_txt.includes('12:00:00')) || entries[Math.floor(entries.length / 2)]
        const temps = entries.map((item) => item.main.temp)
        return { date, label: new Intl.DateTimeFormat('fr-FR', { weekday: 'short', day: 'numeric', month: 'short' }).format(new Date(`${date}T12:00:00`)), icon: noon.weather?.[0]?.icon, description: noon.weather?.[0]?.description, min: Math.round(Math.min(...temps)), max: Math.round(Math.max(...temps)), pop: Math.round(Math.max(...entries.map((item) => item.pop || 0)) * 100) }
      })
    },
    weatherIcon() { return this.current?.weather?.[0]?.icon || '01d' },
    weatherDescription() { return this.current?.weather?.[0]?.description || '' },
  },
  mounted() {
    try { this.history = JSON.parse(localStorage.getItem(HISTORY_KEY) || '[]') } catch { this.history = [] }
    this.history = this.history.filter((item) => item && Number.isFinite(item.lat) && Number.isFinite(item.lon)).slice(0, 5)
    if (this.history.length) this.choisirVille(this.history[0])
    else if (!API_KEY) this.error = 'Ajoutez votre clé OpenWeatherMap dans le fichier .env pour commencer.'
  },
  beforeUnmount() { clearTimeout(this.debounceTimer); this.searchController?.abort() },
  methods: {
    onInputRecherche() {
      clearTimeout(this.debounceTimer); this.searchController?.abort(); this.suggestions = []; this.suggestionError = ''
      const query = this.recherche.trim()
      if (query.length < 2) { this.suggestionsLoading = false; return }
      this.suggestionsLoading = true; this.debounceTimer = setTimeout(() => this.rechercherVilles(query), 400)
    },
    async rechercherVilles(query) {
      if (!API_KEY) { this.suggestionsLoading = false; this.suggestionError = 'Configurez votre clé API dans le fichier .env.'; return }
      const controller = new AbortController(); this.searchController = controller
      try {
        const url = new URL(GEO_URL); url.search = new URLSearchParams({ q: query, limit: '5', appid: API_KEY }).toString()
        const response = await fetch(url, { signal: controller.signal }); const data = await response.json()
        if (!response.ok) throw this.apiError(response.status, data)
        this.suggestions = data.map((place) => ({ ...place, label: [place.name, place.state, place.country].filter(Boolean).join(', ') }))
        if (!this.suggestions.length) this.suggestionError = 'Aucune ville trouvée. Essayez un autre nom.'
      } catch (error) { if (error.name !== 'AbortError') this.suggestionError = error.message }
      finally { if (this.searchController === controller) this.suggestionsLoading = false }
    },
    apiError(status, data) {
      if (status === 401) return new Error('Clé API invalide ou pas encore activée. Vérifiez votre clé OpenWeatherMap.')
      if (status === 429) return new Error('Quota API dépassé. Réessayez un peu plus tard.')
      return new Error(data?.message || `Erreur du service météo (${status}).`)
    },
    async choisirVille(place) {
      if (!API_KEY) { this.error = 'Ajoutez votre clé OpenWeatherMap dans le fichier .env pour commencer.'; return }
      clearTimeout(this.debounceTimer); this.searchController?.abort()
      const id = ++this.requestId
      this.ville = place; this.recherche = ''; this.suggestions = []; this.suggestionError = ''; this.loading = true; this.error = ''
      const historyItem = { name: place.name, state: place.state || '', country: place.country || '', lat: place.lat, lon: place.lon }
      this.history = [historyItem, ...this.history.filter((item) => !(item.lat === place.lat && item.lon === place.lon))].slice(0, 5)
      localStorage.setItem(HISTORY_KEY, JSON.stringify(this.history))
      try {
        const params = new URLSearchParams({ lat: String(place.lat), lon: String(place.lon), appid: API_KEY, units: this.units, lang: 'fr' })
        const [currentResponse, forecastResponse] = await Promise.all([fetch(`${CURRENT_URL}?${params}`), fetch(`${FORECAST_URL}?${params}`)])
        const [currentData, forecastData] = await Promise.all([currentResponse.json(), forecastResponse.json()])
        if (!currentResponse.ok) throw this.apiError(currentResponse.status, currentData)
        if (!forecastResponse.ok) throw this.apiError(forecastResponse.status, forecastData)
        if (id !== this.requestId) return
        this.current = currentData; this.forecast = forecastData.list || []; this.ville = { ...place, name: currentData.name || place.name, country: currentData.sys?.country || place.country }
      } catch (error) { if (id === this.requestId) { this.error = error.message; this.current = null; this.forecast = [] } }
      finally { if (id === this.requestId) this.loading = false }
    },
    async basculerUnite() { this.units = this.units === 'metric' ? 'imperial' : 'metric'; if (this.ville) await this.choisirVille(this.ville) },
    async utiliserPosition() {
      this.error = ''
      if (!API_KEY) { this.error = 'Ajoutez votre clé OpenWeatherMap dans le fichier .env pour commencer.'; return }
      if (!navigator.geolocation) { this.error = 'La géolocalisation n’est pas disponible sur cet appareil.'; return }
      this.loading = true
      navigator.geolocation.getCurrentPosition(async ({ coords }) => { await this.choisirVille({ name: 'Ma position', country: '', lat: coords.latitude, lon: coords.longitude }) }, () => { this.loading = false; this.error = 'Position indisponible. Autorisez la géolocalisation ou recherchez une ville.' }, { timeout: 10000 })
    },
    iconUrl(code, size = '2x') { return `https://openweathermap.org/img/wn/${code || '01d'}@${size}.png` },
    heure(timestamp) { return new Intl.DateTimeFormat('fr-FR', { hour: '2-digit', minute: '2-digit' }).format(new Date(timestamp * 1000)) },
  },
}
</script>

<template>
  <div class="app-shell">
    <header class="header"><a class="logo" href="#"><span class="sun-mark">✳</span> clair<span class="logo-period">.</span></a><div class="header-center"><span class="live-dot"></span> MÉTÉO LOCALE, EN TEMPS RÉEL</div><button class="unit-toggle" @click="basculerUnite" :aria-label="`Passer en degrés ${units === 'metric' ? 'Fahrenheit' : 'Celsius'}`">°{{ units === 'metric' ? 'C' : 'F' }} <span class="toggle-sep">/</span> °{{ units === 'metric' ? 'F' : 'C' }}</button></header>
    <main>
      <section class="intro"><div class="small-label">VOTRE FENÊTRE SUR LE CIEL</div><h1>Le temps, <em>en clair.</em></h1><p>Une ville, un instant. Et le ciel se raconte.</p>
        <div class="search-row"><div class="search-box"><span class="search-glass">⌕</span><input v-model="recherche" @input="onInputRecherche" @keydown.esc="suggestions=[]" placeholder="Quelle ville vous intéresse ?" aria-label="Rechercher une ville" autocomplete="off"><span v-if="suggestionsLoading" class="tiny-spinner"></span><button class="geo-btn" title="Utiliser ma position" aria-label="Utiliser ma position" @click="utiliserPosition">◎</button></div>
          <div v-if="suggestions.length || suggestionError || suggestionsLoading" class="suggestions"><div v-if="suggestionsLoading && !suggestions.length" class="suggestion-note">Recherche des villes…</div><button v-for="place in suggestions" :key="`${place.lat}-${place.lon}`" class="suggestion" @click="choisirVille(place)"><span class="place-pin">◎</span><span>{{ place.label }}</span><span class="suggest-arrow">↗</span></button><div v-if="suggestionError" class="suggestion-note">{{ suggestionError }}</div></div>
        </div>
        <div v-if="history.length" class="history"><span class="history-label">RÉCENT</span><button v-for="item in history" :key="`${item.lat}-${item.lon}`" @click="choisirVille(item)">{{ item.name }}<span v-if="item.country">, {{ item.country }}</span></button></div>
      </section>
      <div v-if="error" class="alert" role="alert"><span>!</span>{{ error }}</div>
      <section v-if="loading" class="loading-card"><span class="loader"></span><strong>On écoute le ciel…</strong><span>Quelques secondes et la météo arrive.</span></section>
      <section v-else-if="current" class="weather-layout">
        <article class="current-card"><div class="current-top"><div><div class="weather-kicker">MAINTENANT <span>·</span> {{ current.sys?.country }}</div><h2>{{ current.name }}<span v-if="current.sys?.country">, {{ current.sys.country }}</span></h2><div class="weather-date">{{ new Intl.DateTimeFormat('fr-FR', { weekday: 'long', day: 'numeric', month: 'long' }).format(new Date()) }}</div></div><img class="current-icon" :src="iconUrl(weatherIcon, '4x')" :alt="weatherDescription"></div>
          <div class="temperature-row"><div class="temperature">{{ Math.round(current.main.temp) }}<sup>{{ unitLabel }}</sup></div><div class="condition">{{ weatherDescription }}<span>Ressenti {{ Math.round(current.main.feels_like) }}{{ unitLabel }}</span></div></div>
          <div class="current-stats"><div><span>HUMIDITÉ</span><strong>{{ current.main.humidity }}<small>%</small></strong></div><div><span>VENT</span><strong>{{ Math.round(current.wind.speed) }}<small> {{ windLabel }}</small></strong></div><div><span>VISIBILITÉ</span><strong>{{ (current.visibility / 1000).toFixed(1) }}<small> km</small></strong></div><div><span>LEVER DU SOLEIL</span><strong>{{ heure(current.sys.sunrise) }}</strong></div></div><div class="coords">{{ current.coord.lat.toFixed(2) }}° N <span>·</span> {{ current.coord.lon.toFixed(2) }}° E</div>
        </article>
        <article class="forecast-card"><div class="section-heading"><div><div class="weather-kicker">LES PROCHAINS JOURS</div><h3>Prévisions à 5 jours</h3></div><span class="forecast-count">3H · {{ groupedForecast.length }} JOURS</span></div><div class="forecast-list"><div v-for="day in groupedForecast" :key="day.date" class="forecast-day"><span class="forecast-date">{{ day.label }}</span><img :src="iconUrl(day.icon)" :alt="day.description"><span class="forecast-desc">{{ day.description }}</span><span class="rain" :class="{ muted: !day.pop }">{{ day.pop ? `◌ ${day.pop}%` : '—' }}</span><span class="temp-range"><b>{{ day.max }}°</b><i></i><span>{{ day.min }}°</span></span></div></div><p class="forecast-foot">Prévisions fournies par OpenWeatherMap, actualisées selon les données disponibles.</p></article>
      </section>
      <section v-else class="empty-sky"><div class="sky-art"><span class="sky-sun">☼</span><span class="sky-cloud">☁</span><span class="sky-star star-one">✦</span><span class="sky-star star-two">✦</span></div><strong>Le ciel attend une adresse.</strong><span>Recherchez une ville pour découvrir sa météo.</span></section>
    </main>
    <footer><span>clair<span class="logo-period">.</span></span><span>FAIT POUR REGARDER DEHORS</span><span>DONNÉES OPENWEATHERMAP</span></footer>
  </div>
</template>
