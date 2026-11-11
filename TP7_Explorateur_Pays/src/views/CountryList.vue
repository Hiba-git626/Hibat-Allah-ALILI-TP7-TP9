<script>
const API_URL = 'https://restcountries.com/v3.1/all?fields=name,flags,capital,region,population,cca3'

export default {
  name: 'CountryList',
  data: () => ({ countries: [], recherche: '', loading: true, error: '' }),
  computed: {
    paysAffiches() {
      const terme = this.recherche.trim().toLocaleLowerCase('fr')
      return this.countries.filter((country) => country.name?.common?.toLocaleLowerCase('fr').includes(terme))
        .sort((a, b) => a.name.common.localeCompare(b.name.common, 'fr'))
    },
  },
  async mounted() { await this.chargerPays() },
  methods: {
    async chargerPays() {
      this.loading = true
      this.error = ''
      try {
        const response = await fetch(API_URL)
        if (!response.ok) throw new Error(`Erreur ${response.status}`)
        this.countries = await response.json()
      } catch (error) {
        this.error = error.message.includes('Failed') || error.message.includes('fetch')
          ? 'Connexion impossible. Vérifiez votre accès internet puis réessayez.'
          : `Impossible de charger les pays (${error.message}).`
      } finally { this.loading = false }
    },
    formatPopulation(value) { return new Intl.NumberFormat('fr-FR').format(value || 0) },
  },
}
</script>

<template>
  <section class="hero">
    <div class="eyebrow"><span class="eyebrow-line"></span> CARNET DE VOYAGE</div>
    <h1>Le monde,<br><em>à portée de clic.</em></h1>
    <p>Explorez les pays, leurs drapeaux et les détails qui font leur singularité.</p>
    <div class="search-wrap"><span class="search-icon">⌕</span><input v-model="recherche" type="search" placeholder="Rechercher un pays…" aria-label="Rechercher un pays"><kbd>⌘ K</kbd></div>
    <div class="count-row"><span>{{ loading ? 'Chargement du globe…' : `${paysAffiches.length} pays trouvés` }}</span><span class="count-line"></span><span class="count-muted">LISTE DES PAYS</span></div>
  </section>

  <section v-if="loading" class="state-card"><span class="spinner"></span><strong>On fait le tour du monde…</strong><span>Les données arrivent</span></section>
  <section v-else-if="error" class="state-card error-card"><span class="state-icon">!</span><strong>Le globe est momentanément indisponible</strong><span>{{ error }}</span><button class="button" @click="chargerPays">Réessayer</button></section>
  <section v-else-if="!paysAffiches.length" class="state-card"><span class="state-icon">⌕</span><strong>Aucun pays trouvé</strong><span>Essayez un autre nom.</span></section>
  <section v-else class="country-grid">
    <RouterLink v-for="country in paysAffiches" :key="country.cca3" class="country-card" :to="`/pays/${country.cca3}`">
      <div class="flag-frame"><img :src="country.flags?.svg || country.flags?.png" :alt="`Drapeau de ${country.name.common}`" loading="lazy"><span class="card-arrow">↗</span></div>
      <div class="card-content"><div class="card-region">{{ country.region || 'Région inconnue' }}</div><h2>{{ country.name.common }}</h2><div class="card-meta"><span>CAPITALE</span><strong>{{ country.capital?.[0] || '—' }}</strong></div><div class="card-meta"><span>POPULATION</span><strong>{{ formatPopulation(country.population) }}</strong></div></div>
    </RouterLink>
  </section>
</template>
