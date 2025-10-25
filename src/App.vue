<template>
  <div id="app">
    <header class="header">
      <div class="container">
        <h1 class="title">
          <span class="title-icon">🏈</span>
          College Football Scoreboard
        </h1>
        <div class="header-info">
          <span class="season-info">{{ seasonInfo }}</span>
          <div class="auto-refresh-indicator" v-if="refreshInterval">
            <span class="auto-refresh-dot"></span>
            Auto-refresh active
          </div>
          <div class="header-buttons">
            <button @click="refreshData" class="refresh-btn" :disabled="manualLoading">
              <span v-if="manualLoading" class="spinner"></span>
              {{ manualLoading ? 'Loading...' : 'Refresh' }}
            </button>
            <button @click="toggleAutoRefresh" class="auto-refresh-btn">
              {{ refreshInterval ? 'Stop Auto' : 'Start Auto' }}
            </button>
          </div>
        </div>
      </div>
    </header>

    <!-- Filters -->
    <div class="filters">
      <div class="container">
        <div class="filter-group">
          <label class="filter-label">Filter by:</label>
          <select v-model="selectedFilter" class="filter-select">
            <option value="all">All Games</option>
            <option value="top25">Top 25 Only</option>
            <option value="sec">SEC</option>
            <option value="big10">Big Ten</option>
            <option value="acc">ACC</option>
            <option value="big12">Big 12</option>
          </select>
        </div>
      </div>
    </div>

    <main class="main">
      <div class="container">
        <div v-if="error" class="error-message">
          <h3>⚠️ Error Loading Data</h3>
          <p>{{ error }}</p>
          <button @click="fetchData" class="retry-btn">Try Again</button>
        </div>

        <div v-else-if="manualLoading && !games.length" class="loading-container">
          <div class="spinner-large"></div>
          <p>Loading games...</p>
        </div>

        <div v-else-if="games.length === 0" class="no-games">
          <h3>No games found</h3>
          <p>There are no games scheduled for this week.</p>
        </div>

        <div v-else class="games-grid">
          <GameCard 
            v-for="game in filteredGames" 
            :key="game.id" 
            :game="game" 
          />
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import axios from 'axios'
import GameCard from './components/GameCard.vue'

export default {
  name: 'App',
  components: {
    GameCard
  },
  setup() {
    const games = ref([])
    const loading = ref(false)
    const manualLoading = ref(false)
    const error = ref(null)
    const seasonData = ref(null)
    const refreshInterval = ref(null)
    const selectedFilter = ref('all')

    const seasonInfo = computed(() => {
      if (!seasonData.value) return ''
      const { season, week } = seasonData.value
      return `${season.year} Season - Week ${week.number}`
    })

    const filteredGames = computed(() => {
      if (!games.value.length) return []
      
      return games.value.filter(game => {
        const competition = game.competitions?.[0]
        if (!competition) return false
        
        const competitors = competition.competitors || []
        const groups = competition.groups
        
        switch (selectedFilter.value) {
          case 'top25':
            return competitors.some(comp => 
              comp.curatedRank && comp.curatedRank.current <= 25
            )
          case 'sec':
            return groups?.id === '8' || groups?.shortName === 'SEC'
          case 'big10':
            return groups?.id === '5' || groups?.shortName === 'Big Ten'
          case 'acc':
            return groups?.id === '1' || groups?.shortName === 'ACC'
          case 'big12':
            return groups?.id === '4' || groups?.shortName === 'Big 12'
          default:
            return true
        }
      })
    })

    const fetchData = async (isManual = false) => {
      if (isManual) {
        manualLoading.value = true
      }
      loading.value = true
      error.value = null
      
      try {
        const response = await axios.get('https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard')
        
        seasonData.value = {
          season: response.data.season,
          week: response.data.week
        }
        
        games.value = response.data.events || []
      } catch (err) {
        error.value = err.message || 'Failed to fetch data'
        console.error('Error fetching data:', err)
      } finally {
        loading.value = false
        if (isManual) {
          manualLoading.value = false
        }
      }
    }

    const refreshData = () => {
      fetchData(true)
    }

    const startAutoRefresh = () => {
      // Clear any existing interval
      if (refreshInterval.value) {
        clearInterval(refreshInterval.value)
      }
      
      // Set up auto-refresh every 30 seconds
      refreshInterval.value = setInterval(() => {
        fetchData()
      }, 30000) // 30 seconds
    }

    const stopAutoRefresh = () => {
      if (refreshInterval.value) {
        clearInterval(refreshInterval.value)
        refreshInterval.value = null
      }
    }

    const toggleAutoRefresh = () => {
      if (refreshInterval.value) {
        stopAutoRefresh()
      } else {
        startAutoRefresh()
      }
    }

    onMounted(() => {
      fetchData()
      startAutoRefresh()
    })

    onUnmounted(() => {
      stopAutoRefresh()
    })

    return {
      games,
      loading,
      manualLoading,
      error,
      seasonInfo,
      refreshInterval,
      selectedFilter,
      filteredGames,
      fetchData,
      refreshData,
      startAutoRefresh,
      stopAutoRefresh,
      toggleAutoRefresh
    }
  }
}
</script>
