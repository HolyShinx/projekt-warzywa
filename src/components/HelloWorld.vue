<template>
  <!-- Description Section -->
  <v-row justify="center">
    <v-col cols="12" md="10" lg="8">
      <v-card class="pa-6 mb-6" elevation="2">
        <v-card-title class="text-h4 text-center green--text text--darken-2 mb-4">
          Projekt Rośliny!
        </v-card-title>
        <v-card-text>
          <p class="text-body-1 mb-4">
            Witamy w naszej kompleksowej bazie danych odmian roślin, Twoim głównym źródle informacji rolniczych i ogrodniczych. 
            Platforma ta zapewnia szczegółowe tabele na temat różnych gatunków roślin, ich cech, warunków wzrostu i różnego typu terminarzy
          </p>
          <p class="text-body-1 mb-4">
            Nasza baza danych zawiera obszerne informacje o różnych odmianach roślin.
            Każdy wpis zawiera kluczowe dane, takie jak pory sadzenia, czasy zbiorów, wymagania glebowe, preferencje klimatyczne i oczekiwania dotyczące plonów.
          </p>
          <p class="text-body-1">
            Niezależnie od tego, czy jesteś profesjonalnym rolnikiem, badaczem rolnictwa czy entuzjastą ogrodnictwa, narzędzie to zawiera naprawdę szeroki obszar zakresu danych.
          </p>
        </v-card-text>
      </v-card>
    </v-col>
  </v-row>

  <!-- Horizontal Divider -->
  <v-row>
    <v-col cols="12">
      <v-divider class="my-6" style="border-color: #4CAF50; border-width: 2px;"></v-divider>
    </v-col>
  </v-row>

  <!-- Loading State -->
  <v-row justify="center" v-if="loading">
    <v-col cols="12" class="text-center">
      <v-progress-circular indeterminate color="green" size="64"></v-progress-circular>
      <p class="mt-4 text-h6">Ładowanie danych roślin...</p>
    </v-col>
  </v-row>

  <!-- Error State -->
  <v-row justify="center" v-if="error">
    <v-col cols="12" md="8">
      <v-alert type="error" class="mb-4">
        <v-alert-title>Błąd ładowania danych</v-alert-title>
        {{ error }}
        <template v-slot:append>
          <v-btn @click="fetchVegetables" color="white" variant="text">
            <v-icon left>mdi-refresh</v-icon>
            Spróbuj ponownie
          </v-btn>
        </template>
      </v-alert>
    </v-col>
  </v-row>

  <!-- Statistics Cards -->
  <v-row justify="center" v-if="!loading && !error">
    <v-col cols="12" md="10" lg="10">
      <v-row class="mb-6">
        <v-col cols="12" sm="6">
          <v-card color="green lighten-4" elevation="2">
            <v-card-text class="text-center">
              <v-icon size="48" color="green darken-2">mdi-sprout-outline</v-icon>
              <div class="text-h4 font-weight-bold mt-2">{{ totalVarieties }}</div>
              <div class="text-subtitle-1">Odmiany ogółem</div>
            </v-card-text>
          </v-card>
        </v-col>
        <v-col cols="12" sm="6">
          <v-card color="light-green lighten-4" elevation="2">
            <v-card-text class="text-center">
              <v-icon size="48" color="light-green darken-2">mdi-leaf</v-icon>
              <div class="text-h4 font-weight-bold mt-2">{{ totalSpecies }}</div>
              <div class="text-subtitle-1">Gatunki roślin</div>
            </v-card-text>
          </v-card>
        </v-col>
      </v-row>
    </v-col>
  </v-row>

  <!-- Filters -->
  <v-row justify="center" v-if="!loading && !error">
    <v-col cols="12" md="10" lg="10">
      <v-card class="mb-4" elevation="1">
        <v-card-text>
          <v-row align="center">
            <v-col cols="12" sm="6" md="6">
              <v-text-field
                v-model="searchQuery"
                label="Szukaj odmiany"
                prepend-icon="mdi-magnify"
                clearable
                variant="outlined"
                density="compact"
              ></v-text-field>
            </v-col>
            <v-col cols="12" sm="6" md="6">
              <v-btn 
                @click="clearFilters" 
                color="grey" 
                variant="outlined"
                class="mr-2"
              >
                <v-icon left>mdi-filter-remove</v-icon>
                Wyczyść filtry
              </v-btn>
              <v-btn 
                @click="fetchVegetables" 
                color="green" 
                variant="outlined"
              >
                <v-icon left>mdi-refresh</v-icon>
                Odśwież
              </v-btn>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
    </v-col>
  </v-row>

  <!-- Table Section -->
  <v-row justify="center" v-if="!loading && !error">
    <v-col cols="12" md="10" lg="10">
      <v-card elevation="2">
        <v-card-title class="text-h5 pa-4 green lighten-5">
          <v-icon left color="green darken-2">mdi-sprout</v-icon>
          Baza danych odmian roślin według gatunków
          <v-spacer></v-spacer>
          <v-chip color="green" text-color="white">
            {{ filteredTableData.length }} wierszy
          </v-chip>
        </v-card-title>
        <v-card-text class="pa-0">
          <v-data-table
            :headers="dynamicHeaders"
            :items="filteredTableData"
            :items-per-page="itemsPerPage"
            class="elevation-0"
            item-key="index"
            show-current-page
            density="compact"
          >
            <template v-for="species in availableSpecies" :key="species" v-slot:[`item.${species}`]="{ item }">
              <div v-if="item[species]" class="pa-2">
                <v-chip 
                  :color="getSpeciesColor(species)" 
                  text-color="white" 
                  small
                  class="ma-1"
                >
                  <v-icon left small>{{ getSpeciesIcon(species) }}</v-icon>
                  {{ item[species] }}
                </v-chip>
              </div>
            </template>
            <template v-slot:no-data>
              <div class="text-center pa-4">
                <v-icon size="64" color="grey">mdi-database-search</v-icon>
                <p class="text-h6 mt-2">Nie znaleziono wyników</p>
                <p class="text-body-2">Spróbuj zmienić kryteria wyszukiwania</p>
              </div>
            </template>
          </v-data-table>
        </v-card-text>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
export default {
  name: 'Home',
  data() {
    return {
      loading: true,
      error: null,
      vegetables: [],
      searchQuery: '',
      itemsPerPage: 10,
      apiUrl: 'http://192.168.1.54:3000/vegetables'
    }
  },
  computed: {
    availableSpecies() {
      return [...new Set(this.vegetables.map(v => v.species))].sort();
    },
    
    dynamicHeaders() {
      return this.availableSpecies.map(species => ({
        title: species,
        key: species,
        align: 'start',
        sortable: false,
        width: `${Math.floor(100 / this.availableSpecies.length)}%`
      }));
    },
    
    groupedBySpecies() {
      const grouped = {};
      this.vegetables.forEach(item => {
        if (!grouped[item.species]) {
          grouped[item.species] = [];
        }
        grouped[item.species].push(item.variety);
      });
      return grouped;
    },
    
    tableData() {
      const maxRows = Math.max(...Object.values(this.groupedBySpecies).map(arr => arr.length));
      const data = [];
      
      for (let i = 0; i < maxRows; i++) {
        const row = { index: i };
        this.availableSpecies.forEach(species => {
          row[species] = this.groupedBySpecies[species] && this.groupedBySpecies[species][i] 
            ? this.groupedBySpecies[species][i] 
            : null;
        });
        data.push(row);
      }
      
      return data;
    },
    
    filteredTableData() {
      if (!this.searchQuery) {
        return this.tableData;
      }
      
      const query = this.searchQuery.toLowerCase();
      return this.tableData.filter(row => {
        return this.availableSpecies.some(species => {
          const value = row[species];
          return value && value.toLowerCase().includes(query);
        });
      });
    },
    
    totalVarieties() {
      return this.vegetables.length;
    },
    
    totalSpecies() {
      return this.availableSpecies.length;
    }
  },
  
  async mounted() {
    await this.fetchVegetables();
  },
  
  methods: {
    async fetchVegetables() {
      this.loading = true;
      this.error = null;

      try {
        const response = await fetch(this.apiUrl);
        
        if (!response.ok) {
          throw new Error(`HTTP Error: ${response.status} ${response.statusText}`);
        }
        
        const data = await response.json();
        
        // Transform API data to match our component structure
        this.vegetables = data.map((item, index) => ({
          id: index + 1,
          species: item['nazwa gatunku'] || 'Nieznany',
          variety: item['nazwa odmiany'] || 'Nieznana'
        }));
        
      } catch (err) {
        console.error('Error fetching vegetables:', err);
        this.error = `Nie udało się pobrać danych: ${err.message}`;
      } finally {
        this.loading = false;
      }
    },
    
    getSpeciesColor(species) {
      const colorMap = {
        'Cebula': 'brown',
        'Kalafior': 'light-green',
        'Marchew': 'orange',
        'Pomidor': 'red',
        'Sałata': 'green',
        'Kapusta': 'teal'
      };
      return colorMap[species] || 'blue-grey';
    },
    
    getSpeciesIcon(species) {
      const iconMap = {
        'Cebula': 'mdi-circle-outline',
        'Kalafior': 'mdi-flower',
        'Marchew': 'mdi-carrot',
        'Pomidor': 'mdi-fruit-cherries',
        'Sałata': 'mdi-leaf',
        'Kapusta': 'mdi-leaf-circle'
      };
      return iconMap[species] || 'mdi-sprout';
    },
    
    clearFilters() {
      this.searchQuery = '';
    }
  }
}
</script>

<style scoped>
.v-data-table >>> .v-data-table__wrapper {
  border-radius: 0;
}

.v-chip {
  font-weight: 500;
}

.font-weight-medium {
  font-weight: 500;
}

.v-data-table >>> .v-data-table-header th {
  background-color: #f5f5f5;
  font-weight: 600;
}
</style>