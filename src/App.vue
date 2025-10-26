<template>
  <div id="app">
    <header class="header">
      <div class="container">
        <h1 class="title">
          <span class="title-icon">🏈</span>
          Scoreboard
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

    <!-- League Tabs -->
    <div class="league-tabs">
      <div class="container">
        <div class="tab-buttons">
          <button 
            @click="setActiveLeague('ncaa')" 
            :class="{ active: activeLeague === 'ncaa' }"
            class="tab-btn"
          >
            <span class="tab-icon">🎓</span>
            NCAA Football
          </button>
          <button 
            @click="setActiveLeague('nfl')" 
            :class="{ active: activeLeague === 'nfl' }"
            class="tab-btn"
          >
            <span class="tab-icon">🏈</span>
            NFL
          </button>
        </div>
      </div>
    </div>

    <!-- Filters -->
    <div class="filters">
      <div class="container">
        <div class="filter-group">
          <label class="filter-label">Filter by:</label>
          <select v-model="selectedFilter" class="filter-select">
            <option v-if="activeLeague === 'ncaa'" value="all">All Games</option>
            <option v-if="activeLeague === 'ncaa'" value="top25">Top 25 Only</option>
            <option v-if="activeLeague === 'ncaa'" value="sec">SEC</option>
            <option v-if="activeLeague === 'ncaa'" value="big10">Big Ten</option>
            <option v-if="activeLeague === 'ncaa'" value="acc">ACC</option>
            <option v-if="activeLeague === 'ncaa'" value="big12">Big 12</option>
            <option v-if="activeLeague === 'nfl'" value="all">All Games</option>
            <option v-if="activeLeague === 'nfl'" value="afc">AFC</option>
            <option v-if="activeLeague === 'nfl'" value="nfc">NFC</option>
            <option v-if="activeLeague === 'nfl'" value="afc-east">AFC East</option>
            <option v-if="activeLeague === 'nfl'" value="afc-west">AFC West</option>
            <option v-if="activeLeague === 'nfl'" value="afc-north">AFC North</option>
            <option v-if="activeLeague === 'nfl'" value="afc-south">AFC South</option>
            <option v-if="activeLeague === 'nfl'" value="nfc-east">NFC East</option>
            <option v-if="activeLeague === 'nfl'" value="nfc-west">NFC West</option>
            <option v-if="activeLeague === 'nfl'" value="nfc-north">NFC North</option>
            <option v-if="activeLeague === 'nfl'" value="nfc-south">NFC South</option>
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
          <NCAAFootballCard 
            v-if="activeLeague === 'ncaa'"
            v-for="game in filteredGames" 
            :key="game.id" 
            :game="game" 
          />
          <NFLGameCard 
            v-if="activeLeague === 'nfl'"
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
import NCAAFootballCard from './components/NCAAFootballCard.vue'
import NFLGameCard from './components/NFLGameCard.vue'

export default {
  name: 'App',
  components: {
    NCAAFootballCard,
    NFLGameCard
  },
  setup() {
    const games = ref([])
    const loading = ref(false)
    const manualLoading = ref(false)
    const error = ref(null)
    const seasonData = ref(null)
    const refreshInterval = ref(null)
    const selectedFilter = ref('all')
    const activeLeague = ref('ncaa')

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
        
        if (activeLeague.value === 'ncaa') {
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
        } else if (activeLeague.value === 'nfl') {
          // NFL filtering logic - show games where at least one team is from the selected conference/division
          switch (selectedFilter.value) {
            case 'afc':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                const isAFC = isAFCTeam(teamName)
                console.log(`AFC filter: ${teamName} (${comp.team?.displayName}) -> ${isAFC}`)
                console.log('Team data:', comp.team)
                return isAFC
              })
            case 'nfc':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                const isNFC = isNFCTeam(teamName)
                console.log(`NFC filter: ${teamName} -> ${isNFC}`)
                return isNFC
              })
            case 'afc-east':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isAFCEastTeam(teamName)
              })
            case 'afc-west':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isAFCWestTeam(teamName)
              })
            case 'afc-north':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isAFCNorthTeam(teamName)
              })
            case 'afc-south':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isAFCSouthTeam(teamName)
              })
            case 'nfc-east':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isNFCEastTeam(teamName)
              })
            case 'nfc-west':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isNFCWestTeam(teamName)
              })
            case 'nfc-north':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isNFCNorthTeam(teamName)
              })
            case 'nfc-south':
              return competitors.some(comp => {
                const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
                return isNFCSouthTeam(teamName)
              })
            default:
              return true
          }
        }
        
        return true
      })
    })

    const fetchData = async (isManual = false) => {
      if (isManual) {
        manualLoading.value = true
      }
      loading.value = true
      error.value = null
      
      try {
        const apiUrl = activeLeague.value === 'ncaa' 
          ? 'https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard'
          : 'https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard'
        
        const response = await axios.get(apiUrl)
        
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

    const setActiveLeague = (league) => {
      activeLeague.value = league
      selectedFilter.value = 'all' // Reset filter when switching leagues
      fetchData(true) // Fetch data for the new league
    }

    // NFL Team Classification Functions
    const isAFCTeam = (teamName) => {
      const afcTeams = ['BUF', 'MIA', 'NE', 'NYJ', 'BAL', 'CIN', 'CLE', 'PIT', 'HOU', 'IND', 'JAX', 'TEN', 'DEN', 'KC', 'LV', 'LAC']
      return afcTeams.includes(teamName.toUpperCase())
    }

    const isNFCTeam = (teamName) => {
      const nfcTeams = ['DAL', 'NYG', 'PHI', 'WAS', 'CHI', 'DET', 'GB', 'MIN', 'ATL', 'CAR', 'NO', 'TB', 'ARI', 'LAR', 'SF', 'SEA']
      return nfcTeams.includes(teamName.toUpperCase())
    }

    const isAFCEastTeam = (teamName) => {
      const afcEastTeams = ['BUF', 'MIA', 'NE', 'NYJ']
      return afcEastTeams.includes(teamName.toUpperCase())
    }

    const isAFCWestTeam = (teamName) => {
      const afcWestTeams = ['DEN', 'KC', 'LV', 'LAC']
      return afcWestTeams.includes(teamName.toUpperCase())
    }

    const isAFCNorthTeam = (teamName) => {
      const afcNorthTeams = ['BAL', 'CIN', 'CLE', 'PIT']
      return afcNorthTeams.includes(teamName.toUpperCase())
    }

    const isAFCSouthTeam = (teamName) => {
      const afcSouthTeams = ['HOU', 'IND', 'JAX', 'TEN']
      return afcSouthTeams.includes(teamName.toUpperCase())
    }

    const isNFCEastTeam = (teamName) => {
      const nfcEastTeams = ['DAL', 'NYG', 'PHI', 'WAS']
      return nfcEastTeams.includes(teamName.toUpperCase())
    }

    const isNFCWestTeam = (teamName) => {
      const nfcWestTeams = ['ARI', 'LAR', 'SF', 'SEA']
      return nfcWestTeams.includes(teamName.toUpperCase())
    }

    const isNFCNorthTeam = (teamName) => {
      const nfcNorthTeams = ['CHI', 'DET', 'GB', 'MIN']
      return nfcNorthTeams.includes(teamName.toUpperCase())
    }

    const isNFCSouthTeam = (teamName) => {
      const nfcSouthTeams = ['ATL', 'CAR', 'NO', 'TB']
      return nfcSouthTeams.includes(teamName.toUpperCase())
    }

    const startAutoRefresh = () => {
      // Clear any existing interval
      if (refreshInterval.value) {
        clearInterval(refreshInterval.value)
      }
      
      // Set up auto-refresh every 20 seconds
      refreshInterval.value = setInterval(() => {
        fetchData()
      }, 20000) // 20 seconds
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
      activeLeague,
      fetchData,
      refreshData,
      setActiveLeague,
      startAutoRefresh,
      stopAutoRefresh,
      toggleAutoRefresh
    }
  }
}
</script>
