<script>
export default {
  name: 'CountryDetail',
  props: { code: { type: String, required: true } },
  data: () => ({ country: null, loading: true, error: '' }),
  computed: {
    languages() { return this.country?.languages ? Object.values(this.country.languages).join(', ') : '—' },
    currencies() { return this.country?.currencies ? Object.values(this.country.currencies).map((c) => `${c.name}${c.symbol ? ` (${c.symbol})` : ''}`).join(', ') : '—' },
  },
  watch: { code: { immediate: true, handler() { this.chargerPays() } } },
  methods: {
    async chargerPays() {
      this.loading = true; this.error = ''; this.country = null
      try {
        const response = await fetch(`https://restcountries.com/v3.1/alpha/${encodeURIComponent(this.code)}?fields=name,flags,capital,region,subregion,population,cca3,cca2,languages,currencies,borders,area,latlng,maps,continents`)
        if (!response.ok) throw new Error(response.status === 404 ? 'Pays introuvable.' : `Erreur ${response.status}`)
        const result = await response.json()
        this.country = Array.isArray(result) ? result[0] : result
      } catch (error) { this.error = error.message.includes('fetch') ? 'Connexion impossible. Vérifiez votre accès internet puis réessayez.' : error.message }
      finally { this.loading = false }
    },
    format(value) { return value == null ? '—' : new Intl.NumberFormat('fr-FR').format(value) },
  },
}
</script>

<template>
  <section class="detail-page">
    <RouterLink to="/" class="back-link">← <span>Retour à l’exploration</span></RouterLink>
    <div v-if="loading" class="state-card detail-state"><span class="spinner"></span><strong>Chargement de la fiche…</strong></div>
    <div v-else-if="error" class="state-card error-card detail-state"><span class="state-icon">!</span><strong>Impossible d’afficher ce pays</strong><span>{{ error }}</span><button class="button" @click="chargerPays">Réessayer</button></div>
    <article v-else-if="country" class="detail-card">
      <div class="detail-heading"><div><div class="eyebrow"><span class="eyebrow-line"></span> {{ country.region || 'MONDE' }}<span v-if="country.subregion"> / {{ country.subregion }}</span></div><h1>{{ country.name.common }}</h1><p class="official-name">{{ country.name.official }}</p></div><div class="detail-code">{{ country.cca3 }}</div></div>
      <div class="detail-flag"><img :src="country.flags?.svg || country.flags?.png" :alt="`Drapeau de ${country.name.common}`"></div>
      <div class="facts-grid">
        <div class="fact"><span>CAPITALE</span><strong>{{ country.capital?.join(', ') || '—' }}</strong></div><div class="fact"><span>POPULATION</span><strong>{{ format(country.population) }}</strong></div>
        <div class="fact"><span>SUPERFICIE</span><strong>{{ format(country.area) }} km²</strong></div><div class="fact"><span>CONTINENT</span><strong>{{ country.continents?.join(', ') || country.region || '—' }}</strong></div>
        <div class="fact"><span>LANGUES</span><strong>{{ languages }}</strong></div><div class="fact"><span>MONNAIE</span><strong>{{ currencies }}</strong></div>
      </div>
      <div v-if="country.borders?.length" class="border-row"><span>PAYS FRONTALIERS</span><div class="border-tags"><RouterLink v-for="border in country.borders" :key="border" :to="`/pays/${border}`">{{ border }}</RouterLink></div></div>
      <a v-if="country.maps?.googleMaps" class="map-link" :href="country.maps.googleMaps" target="_blank" rel="noreferrer">Voir sur Google Maps ↗</a>
    </article>
  </section>
</template>
