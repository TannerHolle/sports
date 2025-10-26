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
            v-for="sport in sports" 
            :key="sport.id"
            @click="setActiveLeague(sport.id)" 
            :class="{ active: activeLeague === sport.id }"
            class="tab-btn"
          >
            <span class="tab-icon">{{ sport.icon }}</span>
            {{ sport.name }}
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
            <option 
              v-for="filter in currentSportFilters" 
              :key="filter.value" 
              :value="filter.value"
            >
              {{ filter.label }}
            </option>
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
          <component 
            :is="currentGameCardComponent"
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
import CollegeBasketballCard from './components/CollegeBasketballCard.vue'

export default {
  name: 'App',
  components: {
    NCAAFootballCard,
    NFLGameCard,
    CollegeBasketballCard
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

    // Sports configuration
    const sports = ref([
      {
        id: 'ncaa',
        name: 'NCAA Football',
        icon: '🎓',
        apiUrl: 'https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard',
        component: 'NCAAFootballCard',
        filters: [
          { value: 'all', label: 'All Games' },
          { value: 'top25', label: 'Top 25 Only' },
          { value: 'sec', label: 'SEC' },
          { value: 'big10', label: 'Big Ten' },
          { value: 'acc', label: 'ACC' },
          { value: 'big12', label: 'Big 12' }
        ]
      },
      {
        id: 'nfl',
        name: 'NFL',
        icon: '🏈',
        apiUrl: 'https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard',
        component: 'NFLGameCard',
        filters: [
          { value: 'all', label: 'All Games' },
          { value: 'afc', label: 'AFC' },
          { value: 'nfc', label: 'NFC' },
          { value: 'afc-east', label: 'AFC East' },
          { value: 'afc-west', label: 'AFC West' },
          { value: 'afc-north', label: 'AFC North' },
          { value: 'afc-south', label: 'AFC South' },
          { value: 'nfc-east', label: 'NFC East' },
          { value: 'nfc-west', label: 'NFC West' },
          { value: 'nfc-north', label: 'NFC North' },
          { value: 'nfc-south', label: 'NFC South' }
        ]
      },
      {
        id: 'ncaa-basketball',
        name: 'NCAA Basketball',
        icon: '🏀',
        apiUrl: 'https://site.api.espn.com/apis/site/v2/sports/basketball/mens-college-basketball/scoreboard',
        component: 'CollegeBasketballCard',
        filters: [
          { value: 'all', label: 'All Games' },
          { value: 'top25', label: 'Top 25 Only' },
          { value: 'acc', label: 'ACC' },
          { value: 'big10', label: 'Big Ten' },
          { value: 'big12', label: 'Big 12' },
          { value: 'sec', label: 'SEC' },
          { value: 'pac12', label: 'Pac-12' },
          { value: 'big-east', label: 'Big East' }
        ]
      }
    ])

    const seasonInfo = computed(() => {
      if (!seasonData.value) return ''
      const { season, week } = seasonData.value
      
      if (activeLeague.value === 'ncaa-basketball') {
        // Basketball shows season without week
        return `${season.year} Season`
      } else if (week) {
        // Football shows season with week
        return `${season.year} Season - Week ${week.number}`
      } else {
        return `${season.year} Season`
      }
    })

    const currentSport = computed(() => {
      return sports.value.find(sport => sport.id === activeLeague.value)
    })

    const currentSportFilters = computed(() => {
      return currentSport.value?.filters || []
    })

    const currentGameCardComponent = computed(() => {
      return currentSport.value?.component || 'NCAAFootballCard'
    })

    const filteredGames = computed(() => {
      if (!games.value.length) return []
      
      return games.value.filter(game => {
        const competition = game.competitions?.[0]
        if (!competition) return false
        
        const competitors = competition.competitors || []
        const groups = competition.groups
        
        // Handle filtering based on current sport
        switch (activeLeague.value) {
          case 'ncaa':
            return filterNCAAGames(competitors, groups, selectedFilter.value)
          case 'nfl':
            return filterNFLGames(competitors, selectedFilter.value)
          case 'ncaa-basketball':
            return filterNCAABasketballGames(competitors, groups, selectedFilter.value)
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
        const apiUrl = currentSport.value?.apiUrl
        
        
        if (!apiUrl) {
          throw new Error('No API URL configured for current sport')
        }
        
        const response = await axios.get(apiUrl)
        
        
        // Handle different data structures for different sports
        try {
          if (activeLeague.value === 'ncaa-basketball') {
            // Basketball doesn't have week data
            seasonData.value = {
              season: response.data.season || { year: new Date().getFullYear() },
              week: null
            }
          } else {
            // Football has week data
            seasonData.value = {
              season: response.data.season || { year: new Date().getFullYear() },
              week: response.data.week || null
            }
          }
        } catch (seasonError) {
          console.error('Error setting season data:', seasonError)
          seasonData.value = {
            season: { year: new Date().getFullYear() },
            week: null
          }
        }
        
        games.value = response.data.events || []
        
      } catch (err) {
        error.value = err.message || 'Failed to fetch data'
        console.error('Error fetching data:', err)
        console.error('Error details:', err.response?.data || err.message)
      } finally {
        loading.value = false
        if (isManual) {
          manualLoading.value = false
        }
        
        // Force clear loading states after a short delay to ensure they're reset
        setTimeout(() => {
          loading.value = false
          manualLoading.value = false
        }, 100)
      }
    }

    const refreshData = () => {
      fetchData(true)
    }

    const setActiveLeague = (league) => {
      activeLeague.value = league
      selectedFilter.value = 'all' // Reset filter when switching leagues
      
      // Force reset loading states
      loading.value = false
      manualLoading.value = false
      
      fetchData(true) // Fetch data for the new league
    }

    // Filtering functions
    const filterNCAAGames = (competitors, groups, filter) => {
      switch (filter) {
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
    }

    const filterNFLGames = (competitors, filter) => {
      switch (filter) {
        case 'afc':
          return competitors.some(comp => {
            const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
            return isAFCTeam(teamName)
          })
        case 'nfc':
          return competitors.some(comp => {
            const teamName = comp.team?.abbreviation || comp.team?.displayName || ''
            return isNFCTeam(teamName)
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

    const filterNCAABasketballGames = (competitors, groups, filter) => {
      switch (filter) {
        case 'top25':
          return competitors.some(comp => 
            comp.curatedRank && comp.curatedRank.current <= 25
          )
        case 'acc':
          return groups?.id === '1' || groups?.shortName === 'ACC'
        case 'big10':
          return groups?.id === '5' || groups?.shortName === 'Big Ten'
        case 'big12':
          return groups?.id === '4' || groups?.shortName === 'Big 12'
        case 'sec':
          return groups?.id === '8' || groups?.shortName === 'SEC'
        case 'pac12':
          return groups?.id === '9' || groups?.shortName === 'Pac-12'
        case 'big-east':
          return groups?.id === '3' || groups?.shortName === 'Big East'
        default:
          return true
      }
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
      sports,
      currentSport,
      currentSportFilters,
      currentGameCardComponent,
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
