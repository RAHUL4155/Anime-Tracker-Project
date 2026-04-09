n<script setup>
import { ref, onMounted, computed, watch } from 'vue'
const exportedData = ref('')

const query = ref('')
const my_anime = ref([])
const search_results = ref([])
const selectedCategory = ref('All')
const isDarkMode = ref(false)
const isLoading = ref(false)
const error = ref(null)
const activeTab = ref('myList')
const showReminderModal = ref(false)
const showExportModal = ref(false)
const showImportModal = ref(false)
const importData = ref('')
const showNoteModal = ref(false)
const currentNoteAnime = ref(null)
const noteContent = ref('')
const showStats = ref(false)


const newReminder = ref({
  animeId: null,
  date: '',
  time: '',
  episode: '',
  repeat: 'none',
  notification: true
})


const filterOptions = ref({
  genre: '',
  year: '',
  status: '',
  language: '',
  rating: ''
})


const genres = ref([
  'Action', 'Adventure', 'Comedy', 'Drama', 
  'Fantasy', 'Horror', 'Mystery', 'Romance',
  'Sci-Fi', 'Slice of Life', 'Sports', 'Supernatural'
])

const languages = ref(['Japanese', 'English', 'Chinese', 'Korean', 'Other'])
const repeatOptions = ref([
  { value: 'none', label: 'No repeat' },
  { value: 'daily', label: 'Daily' },
  { value: 'weekly', label: 'Weekly' },
  { value: 'monthly', label: 'Monthly' }
])

const reminders = ref([])


const today = new Date().toISOString().split('T')[0]


const filteredAnime = computed(() => {
  let list = [...my_anime.value]
  

  list.sort((a, b) => a.title.localeCompare(b.title))
  
  
  if (selectedCategory.value !== 'All') {
    list = list.filter(anime => anime.status === selectedCategory.value)
  }
  

  if (filterOptions.value.genre) {
    list = list.filter(anime => 
      anime.genres?.includes(filterOptions.value.genre)
    )
  }
  
  if (filterOptions.value.year) {
    list = list.filter(anime => anime.year == filterOptions.value.year)
  }
  
  if (filterOptions.value.language) {
    list = list.filter(anime => anime.language === filterOptions.value.language)
  }
  
  if (filterOptions.value.rating) {
    list = list.filter(anime => anime.rating >= filterOptions.value.rating)
  }
  
  return list
})

// Statistics
const animeStats = computed(() => {
  const stats = {
    total: my_anime.value.length,
    watching: my_anime.value.filter(a => a.status === 'Watching').length,
    completed: my_anime.value.filter(a => a.status === 'Completed').length,
    onHold: my_anime.value.filter(a => a.status === 'On Hold').length,
    dropped: my_anime.value.filter(a => a.status === 'Dropped').length,
    planToWatch: my_anime.value.filter(a => a.status === 'Plan to Watch').length,
    totalEpisodes: my_anime.value.reduce((sum, anime) => {
      return sum + (anime.total_episodes === 'Unknown' ? 0 : anime.total_episodes)
    }, 0),
    watchedEpisodes: my_anime.value.reduce((sum, anime) => sum + anime.watched_episodes, 0),
    averageRating: my_anime.value.reduce((sum, anime) => sum + anime.rating, 0) / 
                  my_anime.value.filter(a => a.rating > 0).length || 0
  }
  
  stats.completionPercentage = stats.totalEpisodes > 0 ? 
    Math.round((stats.watchedEpisodes / stats.totalEpisodes) * 100) : 0
    
  return stats
})

// Search anime
const searchAnime = async () => {
  if (!query.value.trim()) {
    error.value = 'Please enter a search term'
    return
  }
  
  try {
    isLoading.value = true
    error.value = null
    const url = `https://api.jikan.moe/v4/anime?q=${query.value}`
    const response = await fetch(url)
    
    if (!response.ok) throw new Error('Failed to fetch anime data')
    
    const data = await response.json()
    search_results.value = data.data || []
  } catch (err) {
    error.value = err.message
    search_results.value = []
  } finally {
    isLoading.value = false
  }
}

// Add anime to list
const addAnime = (anime) => {
  search_results.value = []
  query.value = ''

  my_anime.value.push({
    id: anime.mal_id,
    title: anime.title,
    image: anime.images.jpg.image_url,
    total_episodes: anime.episodes || 'Unknown',
    watched_episodes: 0,
    status: 'Plan to Watch',
    genres: anime.genres?.map(g => g.name) || [],
    year: anime.year || (anime.aired?.prop?.from?.year || new Date().getFullYear()),
    language: 'Japanese', // Default
    rating: 0,
    notes: '',
    added_date: new Date().toISOString().split('T')[0],
    start_date: null,
    finish_date: null
  })

  saveData()
}

// Update watched episodes
const updateWatchedEpisodes = (anime, value) => {
  const newValue = Math.max(0, 
    anime.total_episodes === 'Unknown' ? 
    value : 
    Math.min(value, anime.total_episodes)
  ); 

  // Update start date if this is the first episode watched
  if (anime.watched_episodes === 0 && newValue > 0) {
    anime.start_date = new Date().toISOString().split('T')[0];
  }
  
  // Update finish date if completed
  if (anime.total_episodes !== 'Unknown' && newValue >= anime.total_episodes) {
    anime.finish_date = new Date().toISOString().split('T')[0];
  }
  
  anime.watched_episodes = newValue;
  updateStatus(anime);
  saveData();
}

// Auto-update status based on progress
const updateStatus = (anime) => {
  if (anime.watched_episodes === 0) {
    anime.status = 'Plan to Watch'
  } else if (anime.total_episodes !== 'Unknown' && 
             anime.watched_episodes >= anime.total_episodes) {
    anime.status = 'Completed'
  } else {
    anime.status = 'Watching'
  }
}

// Add/edit note
const saveNote = () => {
  if (currentNoteAnime.value) {
    currentNoteAnime.value.notes = noteContent.value
    saveData()
    showNoteModal.value = false
  }
}

const openNoteModal = (anime) => {
  currentNoteAnime.value = anime
  noteContent.value = anime.notes || ''
  showNoteModal.value = true
}

// Rating system
const rateAnime = (anime, rating) => {
  anime.rating = rating
  saveData()
}

// Add reminder
const addReminder = () => {
  if (!newReminder.value.date || !newReminder.value.animeId) return
  
  const anime = my_anime.value.find(a => a.id === newReminder.value.animeId)
  
  reminders.value.push({
    id: Date.now(),
    animeId: newReminder.value.animeId,
    title: anime.title,
    date: newReminder.value.date,
    time: newReminder.value.time || '12:00',
    episode: newReminder.value.episode || 'Next',
    repeat: newReminder.value.repeat,
    notification: newReminder.value.notification,
    notified: false
  })
  
  saveData()
  showReminderModal.value = false
  resetReminderForm()
}

const resetReminderForm = () => {
  newReminder.value = {
    animeId: null,
    date: '',
    time: '',
    episode: '',
    repeat: 'none',
    notification: true
  }
}

// Check for upcoming reminders
const checkReminders = () => {
  const now = new Date()
  reminders.value.forEach(reminder => {
    const reminderDate = new Date(`${reminder.date}T${reminder.time}`)
    if (!reminder.notified && reminderDate <= now) {
      // Show browser notification if enabled
      if (reminder.notification && Notification.permission === 'granted') {
        new Notification(`Anime Reminder: ${reminder.title}`, {
          body: `Episode ${reminder.episode} is due!`
        })
      }
      
      reminder.notified = true
      
      // Schedule next reminder if repeating
      if (reminder.repeat !== 'none') {
        const newDate = new Date(reminderDate)
        if (reminder.repeat === 'daily') {
          newDate.setDate(newDate.getDate() + 1)
        } else if (reminder.repeat === 'weekly') {
          newDate.setDate(newDate.getDate() + 7)
        } else if (reminder.repeat === 'monthly') {
          newDate.setMonth(newDate.getMonth() + 1)
        }
        
        reminder.date = newDate.toISOString().split('T')[0]
        reminder.notified = false
      }
      
      saveData()
    }
  })
}

// Export/import data
const exportData = () => {
  const data = {
    anime: my_anime.value,
    reminders: reminders.value,
    settings: {
      darkMode: isDarkMode.value
    }
  }
  exportedData.value = JSON.stringify(data, null, 2)
  return exportedData.value
}

const copyToClipboard = () => {
  navigator.clipboard.writeText(exportedData.value)
}

const importFromString = () => {
  try {
    const data = JSON.parse(importData.value)
    if (data.anime) my_anime.value = data.anime
    if (data.reminders) reminders.value = data.reminders
    if (data.settings?.darkMode !== undefined) isDarkMode.value = data.settings.darkMode
    saveData()
    showImportModal.value = false
    importData.value = ''
  } catch (err) {
    error.value = 'Invalid data format'
  }
}

// Request notification permission
const requestNotificationPermission = () => {
  if ('Notification' in window && Notification.permission !== 'granted') {
    Notification.requestPermission()
  }
}

// Data persistence
const saveData = () => {
  try {
    localStorage.setItem('my_anime', JSON.stringify(my_anime.value))
    localStorage.setItem('reminders', JSON.stringify(reminders.value))
    localStorage.setItem('settings', JSON.stringify({
      darkMode: isDarkMode.value
    }))
  } catch (err) {
    console.error('Failed to save data:', err)
  }
}

// Theme management
watch(isDarkMode, (val) => {
  document.body.classList.toggle('dark', val)
  saveData()
})

// Initialize
onMounted(() => {
  try {
    my_anime.value = JSON.parse(localStorage.getItem('my_anime')) || []
    reminders.value = JSON.parse(localStorage.getItem('reminders')) || []
    
    const settings = JSON.parse(localStorage.getItem('settings')) || {}
    isDarkMode.value = settings.darkMode || false
    
    // Check reminders every minute
    setInterval(checkReminders, 60000)
    
    // Request notification permission
    requestNotificationPermission()
  } catch (err) {
    console.error('Failed to load saved data:', err)
  }
})
</script>

<template>
  <main>
    <center><h1 style="color: #1e88e5; font-family: 'Comic Sans MS', 'Segoe UI', cursive;">Anime Tracker</h1></center>

    
    <div class="header-controls">
      <div class="theme-toggle">
        <label class="switch">
          <input type="checkbox" v-model="isDarkMode">
          <span class="slider"></span>
        </label>
        <span class="mode-label">{{ isDarkMode ? '☽ Dark Mode' : '☀️ Light Mode' }}</span>
      </div>

      <div class="utility-buttons">
        <button @click="showExportModal = true" class="button small">
          Export Data
        </button>
        <button @click="showImportModal = true" class="button small">
          Import Data
        </button>
        <button @click="showStats = !showStats" class="button small">
          {{ showStats ? 'Hide Stats' : 'Show Stats' }}
        </button>
      </div>
    </div>

    <!-- Statistics Panel -->
    <div v-if="showStats" class="stats-panel">
      <h3>Your Anime Statistics</h3>
      <div class="stats-grid">
        <div class="stat-card">
          <h4>Total Anime</h4>
          <p>{{ animeStats.total }}</p>
        </div>
        <div class="stat-card">
          <h4>Watching</h4>
          <p>{{ animeStats.watching }}</p>
        </div>
        <div class="stat-card">
          <h4>Completed</h4>
          <p>{{ animeStats.completed }}</p>
        </div>
        <div class="stat-card">
          <h4>On Hold</h4>
          <p>{{ animeStats.onHold }}</p>
        </div>
        <div class="stat-card">
          <h4>Dropped</h4>
          <p>{{ animeStats.dropped }}</p>
        </div>
        <div class="stat-card">
          <h4>Plan to Watch</h4>
          <p>{{ animeStats.planToWatch }}</p>
        </div>
        <div class="stat-card">
          <h4>Total Episodes</h4>
          <p>{{ animeStats.totalEpisodes }}</p>
        </div>
        <div class="stat-card">
          <h4>Watched Episodes</h4>
          <p>{{ animeStats.watchedEpisodes }}</p>
        </div>
        <div class="stat-card">
          <h4>Completion</h4>
          <p>{{ animeStats.completionPercentage }}%</p>
        </div>
        <div class="stat-card">
          <h4>Average Rating</h4>
          <p>{{ animeStats.averageRating.toFixed(1) || 'N/A' }}</p>
        </div>
      </div>
    </div>

    <!-- Navigation Tabs -->
    <div class="tabs">
      <button 
        @click="activeTab = 'myList'" 
        :class="{ active: activeTab === 'myList' }"
      >
        My Anime List
      </button>
      <button 
        @click="activeTab = 'discover'" 
        :class="{ active: activeTab === 'discover' }"
      >
        Discover
      </button>
      <button 
        @click="activeTab = 'reminders'" 
        :class="{ active: activeTab === 'reminders' }"
      >
        Reminders
      </button>
    </div>

    <!-- My Anime List Tab -->
    <div v-if="activeTab === 'myList'" class="tab-content">
      <div v-if="my_anime.length > 0">
        <div class="filters">
          <select v-model="selectedCategory">
            <option>All</option>
            <option>Watching</option>
            <option>Completed</option>
            <option>On Hold</option>
            <option>Dropped</option>
            <option>Plan to Watch</option>
          </select>

          <select v-model="filterOptions.genre">
            <option value="">All Genres</option>
            <option v-for="genre in genres" :value="genre">{{ genre }}</option>
          </select>

          <select v-model="filterOptions.year">
            <option value="">All Years</option>
            <option v-for="year in Array.from({length: 30}, (_, i) => new Date().getFullYear() - i)" 
                    :value="year">
              {{ year }}
            </option>
          </select>

          <select v-model="filterOptions.language">
            <option value="">All Languages</option>
            <option v-for="lang in languages" :value="lang">{{ lang }}</option>
          </select>

          <select v-model="filterOptions.rating">
            <option value="">Any Rating</option>
            <option value="1">1+ Stars</option>
            <option value="2">2+ Stars</option>
            <option value="3">3+ Stars</option>
            <option value="4">4+ Stars</option>
            <option value="5">5 Stars</option>
          </select>
        </div>

        <div class="myanime">
          <div v-for="anime in filteredAnime" :key="anime.id" class="anime">
            <img :src="anime.image" :alt="anime.title" loading="lazy" />
            <div class="anime-details">
              <div class="anime-header">
                <h3>{{ anime.title }}</h3>
                <div class="star-rating">
                  <span v-for="n in 5" 
                        @click="rateAnime(anime, n)"
                        :class="{ filled: n <= anime.rating }">
                    ★
                  </span>
                </div>
              </div>
              
              <div class="anime-meta">
                <span v-if="anime.year">{{ anime.year }}</span>
                <span v-if="anime.genres?.length">{{ anime.genres.join(', ') }}</span>
                <span v-if="anime.language">{{ anime.language }}</span>
              </div>
              
              <p>{{ anime.watched_episodes }} / {{ anime.total_episodes }} episodes</p>
              
              <div class="progress-bar-container" v-if="anime.total_episodes !== 'Unknown'">
                <div class="progress-bar" 
  :style="{ width: ((anime.watched_episodes / anime.total_episodes) * 100) + '%' }">
</div>
              </div>

              <div class="episode-control">
                <button @click="updateWatchedEpisodes(anime, anime.watched_episodes - 1)" 
                        class="button episode-btn">
                  -
                </button>
                <input 
                  type="number" 
                  v-model.number="anime.watched_episodes" 
                  @change="updateWatchedEpisodes(anime, anime.watched_episodes)"
                  min="0"
                  :max="anime.total_episodes === 'Unknown' ? null : anime.total_episodes"
                />
                <button @click="updateWatchedEpisodes(anime, anime.watched_episodes + 1)" 
                        class="button episode-btn">
                  +
                </button>
              </div>

              <div class="anime-dates" v-if="anime.start_date || anime.finish_date">
                <span v-if="anime.start_date">Started: {{ anime.start_date }}</span>
                <span v-if="anime.finish_date">Finished: {{ anime.finish_date }}</span>
              </div>

              <label>Status:
                <select v-model="anime.status" @change="saveData">
                  <option>Watching</option>
                  <option>Completed</option>
                  <option>On Hold</option>
                  <option>Dropped</option>
                  <option>Plan to Watch</option>
                </select>
              </label>

              <div class="actions">
                <button @click="showReminderModal = true; newReminder.animeId = anime.id" 
                        class="button reminder-btn">
                  Set Reminder
                </button>
                <button @click="openNoteModal(anime)" class="button">
                  {{ anime.notes ? 'Edit Note' : 'Add Note' }}
                </button>
                <button @click="my_anime = my_anime.filter(a => a.id !== anime.id); saveData()" 
                        class="button delete">
                  Remove
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-else class="empty-state">
        <p>Your anime list is empty. Search for anime to add to your list!</p>
      </div>
    </div>

    <!-- Discover Tab -->
    <div v-if="activeTab === 'discover'" class="tab-content">
      <form @submit.prevent="searchAnime">
        <input 
          type="text" 
          placeholder="Search for an anime..." 
          v-model="query" 
          @input="handleInput"
          aria-label="Search for anime"
        />
        <button type="submit" class="button" :disabled="isLoading">
          {{ isLoading ? 'Searching...' : 'Search' }}
        </button>
      </form>

      <div v-if="error" class="error-message">
        {{ error }}
      </div>

      <div class="results" v-if="search_results.length > 0">
        <div v-for="anime in search_results" :key="anime.mal_id" class="result">
          <img :src="anime.images.jpg.image_url" :alt="anime.title" loading="lazy" />
          <div class="details">
            <h3>{{ anime.title }}</h3>
            <p :title="anime.synopsis" v-if="anime.synopsis">
              {{ anime.synopsis.slice(0, 120) }}...
            </p>
            <div class="anime-meta">
              <span v-if="anime.year">{{ anime.year }}</span>
              <span v-if="anime.genres">{{ anime.genres.map(g => g.name).join(', ') }}</span>
              <span v-if="anime.episodes">{{ anime.episodes }} episodes</span>
            </div>
            <span class="flex-1"></span>
            <button @click="addAnime(anime)" class="button">Add to My Anime</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Reminders Tab -->
    <div v-if="activeTab === 'reminders'" class="tab-content">
      <button @click="showReminderModal = true" class="button">
        Add New Reminder
      </button>

      <div v-if="reminders.length > 0" class="reminders-list">
        <div v-for="reminder in reminders" :key="reminder.id" class="reminder">
          <div class="reminder-details">
            <h3>{{ reminder.title }} - Episode {{ reminder.episode }}</h3>
            <p>{{ reminder.date }} at {{ reminder.time }}</p>
            <p v-if="reminder.repeat !== 'none'">Repeats: {{ 
              repeatOptions.find(r => r.value === reminder.repeat)?.label 
            }}</p>
          </div>
          <button @click="reminders = reminders.filter(r => r.id !== reminder.id); saveData()" 
                  class="button delete">
            Remove
          </button>
        </div>
      </div>

      <div v-else class="empty-state">
        <p>No reminders set. Add reminders to get notified!</p>
      </div>
    </div>

    <!-- Reminder Modal -->
    <div v-if="showReminderModal" class="modal">
  <div class="modal-content">
    <h3>Add New Reminder</h3>
    
    <div class="form-grid">
      <div class="form-group">
        <label>Anime:</label>
        <select v-model="newReminder.animeId" required class="form-control">
          <option v-for="anime in my_anime" :value="anime.id">
            {{ anime.title }}
          </option>
        </select>
      </div>
      
      <div class="form-group">
        <label>Date:</label>
        <input type="date" v-model="newReminder.date" :min="today" required class="form-control">
      </div>
      
      <div class="form-group">
        <label>Time:</label>
        <input type="time" v-model="newReminder.time" class="form-control">
      </div>
      
      <div class="form-group">
        <label>Episode:</label>
        <input 
          type="text" 
          v-model="newReminder.episode" 
          placeholder="Next episode or specific number" 
          class="form-control"
        >
      </div>
      
      <div class="form-group">
        <label>Repeat:</label>
        <select v-model="newReminder.repeat" class="form-control">
          <option v-for="option in repeatOptions" :value="option.value">
            {{ option.label }}
          </option>
        </select>
      </div>
      
      <div class="form-group checkbox-group">
        <label class="checkbox-label">
          <input type="checkbox" v-model="newReminder.notification">
          Browser Notification
        </label>
      </div>
    </div>
    
    <div class="modal-actions">
      <button @click="addReminder" class="button">Save Reminder</button>
      <button @click="showReminderModal = false" class="button cancel">Cancel</button>
    </div>
  </div>
</div>

    <!-- Note Modal -->
    <div v-if="showNoteModal" class="modal">
      <div class="modal-content">
        <h3>Note for {{ currentNoteAnime?.title }}</h3>
        <textarea v-model="noteContent" rows="8"></textarea>
        <div class="modal-actions">
          <button @click="saveNote" class="button">Save Note</button>
          <button @click="showNoteModal = false" class="button cancel">Cancel</button>
        </div>
      </div>
    </div>

    <!-- Export Modal -->
    <div v-if="showExportModal" class="modal">
  <div class="modal-content">
    <h3>Export Your Data</h3>
    <p>Copy this data to save or transfer it to another device:</p>
    <textarea readonly rows="10" :value="exportedData"></textarea>
    <div class="modal-actions">
      <button @click="copyToClipboard" class="button">Copy to Clipboard</button>
      <button @click="showExportModal = false" class="button cancel">Close</button>
    </div>
  </div>
</div>

    <!-- Import Modal -->
    <div v-if="showImportModal" class="modal">
      <div class="modal-content">
        <h3>Import Data</h3>
        <p>Paste your exported data here:</p>
        <textarea v-model="importData" rows="10"></textarea>
        <div v-if="error" class="error-message">{{ error }}</div>
        <div class="modal-actions">
          <button @click="importFromString" class="button">Import</button>
          <button @click="showImportModal = false" class="button cancel">Cancel</button>
        </div>
      </div>
    </div>
  </main>
</template>

<style>
/* Base styles */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Fira Sans', sans-serif;
}

body {
  background-color: #EEE;
  color: #333;
  transition: background-color 0.3s, color 0.3s;
}

body.dark {
  background-color: #121212;
  color: #FFF;
}

main {
  margin: 0 auto;
  max-width: 1200px;
  padding: 1.5rem;
}

h1, h2, h3, h4 {
  margin-bottom: 1rem;
  color: inherit;
}

.button {
  appearance: none;
  outline: none;
  border: none;
  cursor: pointer;
  padding: 0.5rem 1rem;
  background-image: linear-gradient(to right, #1e88e5, #0d47a1);
  background-size: 200%;
  color: white;
  font-size: 1rem;
  font-weight: bold;
  transition: 0.4s;
  border-radius: 0.25rem;
  margin: 0.5rem 0;
}

.button:hover {
  background-position: right;
}

.button.small {
  padding: 0.3rem 0.7rem;
  font-size: 0.875rem;
}

.button.delete {
  background-image: linear-gradient(to right, #1e88e5, #0d47a1);
}

.button.cancel {
  background-image: linear-gradient(to right, #666, #444);
}

/* Header controls */
.header-controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.theme-toggle {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.utility-buttons {
  display: flex;
  gap: 0.5rem;
}

/* Switch styles (same as before) */
.switch {
  position: relative;
  display: inline-block;
  width: 60px;
  height: 34px;
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
  transition: .4s;
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
  transition: .4s;
  border-radius: 50%;
}

input:checked + .slider {
  background-color: #1e88e5;
}

input:checked + .slider:before {
  transform: translateX(26px);
}

/* Stats panel */
.stats-panel {
  background-color: #f8f8f8;
  border-radius: 0.5rem;
  padding: 1rem;
  margin-bottom: 1.5rem;
}

body.dark .stats-panel {
  background-color: #2c2c2c;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 1rem;
  margin-top: 1rem;
}

.stat-card {
  background-color: white;
  border-radius: 0.5rem;
  padding: 1rem;
  text-align: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

body.dark .stat-card {
  background-color: #1e1e1e;
}

.stat-card h4 {
  font-size: 0.875rem;
  margin-bottom: 0.5rem;
  color: #666;
}

body.dark .stat-card h4 {
  color: #AAA;
}

.stat-card p {
  font-size: 1.5rem;
  font-weight: bold;
}

/* Tabs */
.tabs {
  display: flex;
  margin-bottom: 1.5rem;
  border-bottom: 1px solid #ccc;
}

body.dark .tabs {
  border-bottom-color: #444;
}

.tabs button {
  padding: 0.75rem 1.5rem;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1rem;
  border-bottom: 3px solid transparent;
  transition: all 0.3s;
  color: inherit;
}

.tabs button.active {
  border-bottom-color: #1e88e5;
  font-weight: bold;
}

/* Filters */
.filters {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.filters select, .filters input {
  flex: 1;
  min-width: 150px;
  padding: 0.5rem;
  border-radius: 0.25rem;
  border: 1px solid #ccc;
}

body.dark .filters select,
body.dark .filters input {
  background-color: #2c2c2c;
  color: white;
  border-color: #444;
}

/* Anime list */
.myanime {
  display: grid;
  gap: 1rem;
}

.anime {
  display: flex;
  background-color: white;
  border-radius: 0.5rem;
  overflow: hidden;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

body.dark .anime {
  background-color: #1e1e1e;
}

.anime img {
  width: 120px;
  height: 170px;
  object-fit: cover;
}

.anime-details {
  flex: 1;
  padding: 1rem;
  display: flex;
  flex-direction: column;
}

.anime-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.star-rating {
  color: #ccc;
  font-size: 1.5rem;
  cursor: pointer;
}

.star-rating .filled {
  color: #1e88e5;
}

.anime-meta {
  display: flex;
  gap: 1rem;
  font-size: 0.875rem;
  color: #666;
  margin: 0.5rem 0;
  flex-wrap: wrap;
}

body.dark .anime-meta {
  color: #AAA;
}

.anime-meta span {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.anime-dates {
  display: flex;
  gap: 1rem;
  font-size: 0.875rem;
  color: #666;
  margin: 0.5rem 0;
}

body.dark .anime-dates {
  color: #AAA;
}

/* Progress bar */
.progress-bar-container {
  height: 10px;
  background-color: #ddd;
  border-radius: 0.5rem;
  margin: 0.5rem 0;
  overflow: hidden;
}

body.dark .progress-bar-container {
  background-color: #444;
}

.progress-bar {
  height: 100%;
  background-image: linear-gradient(to right, #1e88e5, #0d47a1);
  transition: width 0.3s ease;
}

/* Episode controls */
.episode-control {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin: 0.5rem 0;
}

.episode-control input {
  width: 60px;
  padding: 0.5rem;
  text-align: center;
  border: 1px solid #ccc;
  border-radius: 0.25rem;
}

body.dark .episode-control input {
  background-color: #2c2c2c;
  color: white;
  border-color: #444;
}

.episode-btn {
  width: 40px;
  padding: 0.5rem;
}

/* Actions */
.actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.5rem;
  flex-wrap: wrap;
}

/* Results (search) */
.results {
  display: grid;
  gap: 1rem;
  margin-top: 1rem;
}

.result {
  display: flex;
  background-color: white;
  border-radius: 0.5rem;
  overflow: hidden;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

body.dark .result {
  background-color: #1e1e1e;
}

.result img {
  width: 100px;
  height: 140px;
  object-fit: cover;
}

.details {
  flex: 1;
  padding: 1rem;
  display: flex;
  flex-direction: column;
}


.reminders-list {
  display: grid;
  gap: 0.5rem;
  margin-top: 1rem;
}

.reminder {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #f8f8f8;
  padding: 1rem;
  border-radius: 0.5rem;
}

body.dark .reminder {
  background-color: #2c2c2c;
}

.reminder-details {
  flex: 1;
}


.modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}

.modal-content {
  background-color: white;
  padding: 2rem;
  border-radius: 0.5rem;
  width: 100%;
  max-width: 500px;
  max-height: 90vh;
  overflow-y: auto;
}

body.dark .modal-content {
  background-color: #1e1e1e;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
  margin-top: 1rem;
}

textarea {
  width: 100%;
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 0.25rem;
  margin: 0.5rem 0;
}

body.dark textarea {
  background-color: #2c2c2c;
  color: white;
  border-color: #444;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin: 0.5rem 0;
}


.empty-state {
  text-align: center;
  padding: 2rem;
  color: #666;
}

body.dark .empty-state {
  color: #AAA;
}


.error-message {
  color: #ff4444;
  padding: 0.5rem;
  margin: 0.5rem 0;
  background-color: rgba(255, 68, 68, 0.1);
  border-radius: 0.25rem;
}

body.dark .error-message {
  background-color: rgba(255, 68, 68, 0.2);
}


@media (max-width: 768px) {
  .header-controls {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .tabs button {
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
  }
  
  .anime, .result {
    flex-direction: column;
  }
  
  .anime img, .result img {
    width: 100%;
    height: auto;
    max-height: 200px;
  }
  
  .modal-content {
    margin: 1rem;
    padding: 1rem;
  }
  
  .stats-grid {
    grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  }
}

@media (max-width: 480px) {
  .filters {
    flex-direction: column;
  }
  
  .actions {
    flex-direction: column;
  }
  
  .button {
    width: 100%;
  }
}
/* Modal styles */
.modal-content {
  max-width: 500px;
  padding: 2rem;
}

.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
  margin: 1.5rem 0;
}

.form-group {
  display: flex;
  flex-direction: column;
  margin-bottom: 0.5rem;
}

.form-group label {
  margin-bottom: 0.5rem;
  font-weight: 500;
  color: #333;
}

body.dark .form-group label {
  color: #eee;
}

.form-control {
  padding: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
}

body.dark .form-control {
  background-color: #2c2c2c;
  color: white;
  border-color: #444;
}

.checkbox-group {
  grid-column: span 2;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
  margin-top: 1rem;
}


@media (max-width: 600px) {
  .form-grid {
    grid-template-columns: 1fr;
  }
  
  .checkbox-group {
    grid-column: span 1;
  }
}
</style>
