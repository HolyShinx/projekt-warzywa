<template>
  <v-container>
    <v-card>
      <v-card-title class="custom-card-title">
        Kapusta - Charakterystyka odmian
      </v-card-title>
      
      <!-- Loading indicator -->
      <v-progress-linear 
        v-if="loading" 
        indeterminate 
        color="warning"
      ></v-progress-linear>
      
      <!-- Error message -->
      <v-alert 
        v-if="error" 
        type="error" 
        dismissible 
        @click:close="error = null"
        class="ma-4"
      >
        {{ error }}
      </v-alert>
      
      <!-- Deletion mode info -->
      <v-alert
        v-if="entryMode === 'delete'"
        type="warning"
        variant="tonal"
        class="ma-4"
        prominent
      >
        <template v-slot:prepend>
          <v-icon>mdi-delete-alert</v-icon>
        </template>
        <strong>Tryb Usuwania:</strong> Kliknij na odmianę w tabeli aby ją usunąć. Odmiany nie wygenerowane przez AI będą wymagały potwierdzenia.
      </v-alert>
      
      <div class="table-wrapper">
        <v-data-table
          :headers="headers"
          :items="varieties"
          item-key="nazwa odmiany"
          class="elevation-1"
          :items-per-page="-1"
          hide-default-footer
          :loading="loading"
        >
          
          <!-- Custom slots for symbol columns -->
          <template v-slot:item.odpornosc="{ item }">
            <span class="odpornosc-symbols">{{ getResistanceSymbols(item['odporność na pękanie']) }}</span>
          </template>
          
          <template v-slot:item.przetworstwo="{ item }">
            <span class="przeznaczenie-symbols">{{ getBooleanSymbol(item['przetwórstwo']) }}</span>
          </template>
          
          <template v-slot:item.przechowywanie="{ item }">
            <span class="przeznaczenie-symbols">{{ getBooleanSymbol(item.przechowywanie) }}</span>
          </template>
          
          <template v-slot:item.swiezyRynek="{ item }">
            <span class="przeznaczenie-symbols">{{ getBooleanSymbol(item['świeży rynek']) }}</span>
          </template>

          <!-- Custom template for odmiana column with delete functionality -->
          <template v-slot:item.odmiana="{ item }">
            <span 
              :class="{ 
                'generated-variety': item.generowane, 
                'normal-variety': !item.generowane,
                'deletable-row': entryMode === 'delete'
              }"
              @click="entryMode === 'delete' ? handleDeleteClick(item) : null"
            >
              {{ item['nazwa odmiany'] }}
              <v-icon 
                v-if="item.generowane" 
                size="small" 
                class="ml-1 generated-icon"
              >
                mdi-auto-fix
              </v-icon>
              <v-icon 
                v-if="entryMode === 'delete'" 
                size="small" 
                class="ml-1 delete-icon"
              >
                mdi-delete
              </v-icon>
            </span>
          </template>

          <!-- Make all table rows clickable in delete mode -->
          <template v-slot:item="{ item }">
            <tr 
              :class="{ 'deletable-table-row': entryMode === 'delete' }"
              @click="entryMode === 'delete' ? handleDeleteClick(item) : null"
            >
              <td class="text-center">
                <span 
                  :class="{ 
                    'generated-variety': item.generowane, 
                    'normal-variety': !item.generowane,
                    'deletable-row': entryMode === 'delete'
                  }"
                >
                  {{ item['nazwa odmiany'] }}
                  <v-icon 
                    v-if="item.generowane" 
                    size="small" 
                    class="ml-1 generated-icon"
                  >
                    mdi-auto-fix
                  </v-icon>
                  <v-icon 
                    v-if="entryMode === 'delete'" 
                    size="small" 
                    class="ml-1 delete-icon"
                  >
                    mdi-delete
                  </v-icon>
                </span>
              </td>
              <td class="text-center">{{ item['gęstość nasadzenia'] }}</td>
              <td class="text-center">{{ item['okres wegetacji'] }}</td>
              <td class="text-center">
                <span class="odpornosc-symbols">{{ getResistanceSymbols(item['odporność na pękanie']) }}</span>
              </td>
              <td class="text-center">{{ item.masa }}</td>
              <td class="text-center">
                <span class="przeznaczenie-symbols">{{ getBooleanSymbol(item['przetwórstwo']) }}</span>
              </td>
              <td class="text-center">
                <span class="przeznaczenie-symbols">{{ getBooleanSymbol(item.przechowywanie) }}</span>
              </td>
              <td class="text-center">
                <span class="przeznaczenie-symbols">{{ getBooleanSymbol(item['świeży rynek']) }}</span>
              </td>
              <td class="text-center">{{ item.uwagi }}</td>
            </tr>
          </template>
        </v-data-table>
      </div>
    </v-card>

    <!-- Confirmation Dialog -->
    <v-dialog v-model="showDeleteDialog" max-width="500px">
      <v-card>
        <v-card-title class="text-h5">
          <v-icon color="warning" class="mr-2">mdi-alert</v-icon>
          Potwierdź usunięcie
        </v-card-title>
        <v-card-text>
          Czy na pewno chcesz usunąć odmianę <strong>"{{ itemToDelete?.['nazwa odmiany'] }}"</strong>?
          <br><br>
          <span class="text-caption text-medium-emphasis">
            Ta operacja jest nieodwracalna.
          </span>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn 
            color="grey" 
            variant="text" 
            @click="cancelDelete"
            :disabled="deleting"
          >
            Anuluj
          </v-btn>
          <v-btn 
            color="red" 
            variant="elevated"
            @click="confirmDelete"
            :loading="deleting"
          >
            <v-icon left>mdi-delete</v-icon>
            Usuń
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Mode Toggle Button -->
    <div class="mt-4 text-center">
      <v-btn-toggle
        v-model="entryMode"
        mandatory
        variant="outlined"
        divided
        class="mode-toggle"
      >
        <v-btn value="simple" color="primary">
          <v-icon left>mdi-auto-fix</v-icon>
          Tryb Szybki
        </v-btn>
        <v-btn value="full" color="secondary">
          <v-icon left>mdi-table-edit</v-icon>
          Tryb Pełny
        </v-btn>
        <v-btn value="delete" color="error">
          <v-icon left>mdi-delete</v-icon>
          Tryb Usuwania
        </v-btn>
      </v-btn-toggle>
    </div>

    <!-- Data Entry Table -->
    <v-card class="mt-4" v-if="entryMode !== 'delete'">
      <v-card-title class="custom-card-title">
        Dodaj nową odmianę kapusty
      </v-card-title>
      
      <!-- Simple Mode (AI Generation) -->
      <div v-if="entryMode === 'simple'" class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Odmiana</th>
              <th class="entry-header">Cechy Charakterystyczne</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td class="entry-cell first-column">
                <v-text-field
                  v-model="newVariety['nazwa odmiany']"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety.uwagi"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
            </tr>
          </tbody>
        </v-table>
        
        <div class="mt-3 text-center">
          <v-btn
            color="success"
            @click="generateVariety"
            :disabled="!newVariety['nazwa odmiany'] || !newVariety.uwagi || generating"
            :loading="generating"
            class="mr-2"
          >
            <v-icon left size="small">mdi-auto-fix</v-icon>
            Generuj
          </v-btn>
          <v-btn
            color="secondary"
            @click="clearForm"
            variant="outlined"
            :disabled="generating"
          >
            Wyczyść
          </v-btn>
          <v-btn
            color="primary"
            @click="fetchVarieties"
            variant="outlined"
            :loading="loading"
            class="ml-2"
          >
            Odśwież dane
          </v-btn>
        </div>
        
        <!-- Info message for AI generation -->
        <v-alert
          v-if="showGenerateInfo"
          type="info"
          variant="tonal"
          class="mt-3"
          closable
          @click:close="showGenerateInfo = false"
        >
          <strong>Tryb AI:</strong> Wprowadź nazwę odmiany i cechy charakterystyczne, a reszta będzie wygenerowana automatycznie.
        </v-alert>
      </div>

      <!-- Full Mode (Manual Entry) -->
      <div v-else class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Odmiana</th>
              <th class="entry-header">Gęstość nasadzenia</th>
              <th class="entry-header">Okres wegetacji</th>
              <th class="entry-header">Odporność na pękanie</th>
              <th class="entry-header">Średnia masa (kg)</th>
              <th class="entry-header">Przetwórstwo</th>
              <th class="entry-header">Przechowywanie</th>
              <th class="entry-header">Świeży Rynek</th>
              <th class="entry-header">Cechy</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td class="entry-cell first-column">
                <v-text-field
                  v-model="newVariety['nazwa odmiany']"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety['gęstość nasadzenia']"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety['okres wegetacji']"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety['odporność na pękanie']"
                  :items="odpornoscOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety.masa"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety['przetwórstwo']"
                  :items="przeznczenieOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.przechowywanie"
                  :items="przeznczenieOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety['świeży rynek']"
                  :items="przeznczenieOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety.uwagi"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
            </tr>
          </tbody>
        </v-table>
        
        <div class="mt-3 text-center">
          <v-btn
            color="warning"
            @click="addVariety"
            :disabled="!isFullDataAvailable || submitting"
            :loading="submitting"
            class="mr-2"
          >
            Dodaj odmianę
          </v-btn>
          <v-btn
            color="secondary"
            @click="clearForm"
            variant="outlined"
            :disabled="submitting"
          >
            Wyczyść
          </v-btn>
          <v-btn
            color="primary"
            @click="fetchVarieties"
            variant="outlined"
            :loading="loading"
            class="ml-2"
          >
            Odśwież dane
          </v-btn>
        </div>
        
        <!-- Info message for full mode -->
        <v-alert
          type="info"
          variant="tonal"
          class="mt-3"
        >
          <strong>Tryb Pełny:</strong> Wypełnij wszystkie pola aby aktywować przycisk "Dodaj odmianę".
        </v-alert>
      </div>
    </v-card>
    
    <!-- Bulk Generation Section -->
    <v-card class="mt-4" v-if="entryMode !== 'delete'">
      <v-card-title class="custom-card-title">
        Generuj wiele odmian naraz
      </v-card-title>
      
      <div class="pa-4">
        <div class="d-flex align-center justify-center gap-3">
          <v-number-input
            v-model="bulkCount"
            :min="1"
            :max="50"
            variant="outlined"
            density="compact"
            label="Liczba odmian"
            style="width: 200px;"
            hide-details
          ></v-number-input>
          
          <v-btn
            color="success"
            @click="generateManyVarieties"
            :disabled="!bulkCount || bulkCount < 1 || generatingMany"
            :loading="generatingMany"
            size="large"
          >
            <v-icon left size="small">mdi-auto-fix</v-icon>
            Generuj {{ bulkCount || 0 }} odmian
          </v-btn>
        </div>
        
        <v-alert
          type="info"
          variant="tonal"
          class="mt-3"
        >
          <strong>Generowanie masowe:</strong> Wybierz liczbę odmian (1-50) do automatycznego wygenerowania przez AI.
        </v-alert>
      </div>
    </v-card>
  </v-container>
</template>

<script>
export default {
  name: 'CabbageVarietiesTable',
  data() {
    return {
      loading: false,
      error: null,
      submitting: false,
      generating: false,
      generatingMany: false,
      deleting: false,
      showGenerateInfo: true,
      showDeleteDialog: false,
      itemToDelete: null,
      entryMode: 'simple', // 'simple', 'full', or 'delete'
      bulkCount: 5, // Default bulk generation count
      
      // API endpoints - replace with your actual cabbage API endpoints
      apiUrl: 'http://192.168.1.54:3000/white-cabbage',
      addApiUrl: 'http://192.168.1.54:3000/white-cabbage/add',
      generateApiUrl: 'http://192.168.1.54:3000/white-cabbage/generate',
      generateManyApiUrl: 'http://192.168.1.54:3000/white-cabbage/generate-many',
      deleteApiUrl: 'http://192.168.1.54:3000/white-cabbage/by-name/VarietyName',
      
      headers: [
        {
          title: 'Odmiana',
          value: 'nazwa odmiany', // Changed from 'odmiana'
          align: 'start',
          sortable: true,
          fixed: true,
          width: '200px',
          class: 'odmiana-column',
        },
        { title: 'Gęstość nasadzenia', key: 'gęstość nasadzenia', sortable: true, width: '150px' }, // Changed from 'gestosc'
        { title: 'Okres wegetacji', key: 'okres wegetacji', sortable: true, width: '150px' }, // Changed from 'okresWegetacji'
        { 
          title: 'Główki', 
          key: 'glowki', 
          sortable: false, 
          width: '300px',
          class: 'owoc-header-custom',
          children: [
            { 
              title: 'Odporność na pękanie', 
              key: 'odporność na pękanie', // Changed from 'odpornosc'
              sortable: true, 
              width: '150px',
              class: 'masa-column'
            },
            { 
              title: 'Średnia masa (kg)', 
              key: 'masa', 
              sortable: true, 
              width: '150px',
              class: 'owoc-child-header' 
            },
          ]
        },
        { 
          title: 'Przeznaczenie', 
          key: 'przeznaczenie', 
          sortable: false, 
          width: '450px',
          class: 'owoc-header-custom',
          children: [
            { 
              title: 'Przetwórstwo', 
              key: 'przetwórstwo', // Changed from 'przetworstwo'
              sortable: true, 
              width: '150px',
              class: 'masa-column' 
            },
            { 
              title: 'Przechowywanie', 
              key: 'przechowywanie', 
              sortable: true, 
              width: '150px',
              class: 'owoc-child-header' 
            },
            { 
              title: 'Świeży Rynek', 
              key: 'świeży rynek', // Changed from 'swiezyRynek'
              sortable: true, 
              width: '150px',
              class: 'masa-column' 
            },
          ]
        },
        { title: 'Cecha Charakterystyczna', key: 'uwagi', sortable: true, width: '180px' }, // Changed from 'cechy'
      ],  
      varieties: [
        // Default varieties from your original data - updated to match JSON structure
        {
          'nazwa odmiany': 'Herkules',
          'gęstość nasadzenia': '40-50 tys./ha',
          'okres wegetacji': '90-110 dni',
          'odporność na pękanie': 3, // Convert to numeric for proper symbol display
          masa: '2-4 kg',
          przechowywanie: true,
          'przetwórstwo': true,
          'świeży rynek': true,
          uwagi: 'Bardzo plenna, dobra do długiego przechowywania.',
          generowane: false
        },
        {
          'nazwa odmiany': 'Wolska',
          'gęstość nasadzenia': '35-45 tys./ha',
          'okres wegetacji': '120-140 dni',
          'odporność na pękanie': 2,
          masa: '3-5 kg',
          przechowywanie: true,
          'przetwórstwo': false,
          'świeży rynek': false,
          uwagi: 'Klasyczna odmiana, twarda.',
          generowane: false
        },
        {
          'nazwa odmiany': 'Red Baron',
          'gęstość nasadzenia': '40-50 tys./ha',
          'okres wegetacji': '100-120 dni',
          'odporność na pękanie': 2,
          masa: '1.5-3 kg',
          przechowywanie: false,
          'przetwórstwo': false,
          'świeży rynek': true,
          uwagi: 'Intensywna barwa, łagodniejsza w smaku.',
          generowane: false
        }
      ],
      newVariety: {
        'nazwa odmiany': '',
        'gęstość nasadzenia': '',
        'okres wegetacji': '',
        'odporność na pękanie': 1,
        masa: '',
        'przetwórstwo': false,
        przechowywanie: false,
        'świeży rynek': false,
        uwagi: '',
        generowane: false
      },
      odpornoscOptions: [
        { title: '■ (Niska)', value: 1 },
        { title: '■■ (Średnia)', value: 2 },
        { title: '■■■ (Wysoka)', value: 3 }
      ],
      przeznczenieOptions: [
        { title: 'Nie', value: false },
        { title: 'Tak', value: true }
      ]
    };
  },
  computed: {
    isFullDataAvailable() {
      return this.newVariety['nazwa odmiany'] && 
             this.newVariety['gęstość nasadzenia'] && 
             this.newVariety['okres wegetacji'] && 
             this.newVariety['odporność na pękanie'] && 
             this.newVariety.masa && 
             this.newVariety.uwagi;
    }
  },
  async mounted() {
    await this.fetchVarieties();
  },  
  methods: {
    // Helper method to convert boolean to symbols for display
    getBooleanSymbol(value) {
      return value ? '■■■' : '■';
    },
    
    // Helper method to convert numeric resistance to symbols
    getResistanceSymbols(level) {
      const symbols = {
        1: '■',
        2: '■■', 
        3: '■■■'
      };
      return symbols[level] || '■';
    },
    
    async fetchVarieties() {
      this.loading = true;
      this.error = null;
      
      try {
        const response = await fetch(this.apiUrl);
        
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const data = await response.json();
        this.varieties = Array.isArray(data) ? data : [];
        
      } catch (error) {
        console.error('Error fetching varieties:', error);
        this.error = `Błąd podczas pobierania danych: ${error.message}`;
        
        // Fallback to empty array if API fails
        this.varieties = [];
      } finally {
        this.loading = false;
      }
    },

    handleDeleteClick(item) {
      if (this.entryMode !== 'delete') return;
      
      this.itemToDelete = item;
      
      // If it's a generated variety, delete immediately
      if (item.generowane) {
        this.confirmDelete();
      } else {
        // Show confirmation dialog for non-generated varieties
        this.showDeleteDialog = true;
      }
    },

    cancelDelete() {
      this.showDeleteDialog = false;
      this.itemToDelete = null;
    },

    async confirmDelete() {
      if (!this.itemToDelete) return;
      
      this.deleting = true;
      this.error = null;
      
      try {
        // Get the variety name from the item
        const varietyName = this.itemToDelete['nazwa odmiany'];
        
        // Construct the URL by replacing the template parameter with the actual variety name
        const deleteUrl = this.deleteApiUrl.replace('VarietyName', encodeURIComponent(varietyName));
        
        console.log('Delete URL:', deleteUrl); // Debug log
        console.log('Deleting variety:', varietyName); // Debug log
        
        const response = await fetch(deleteUrl, {
          method: 'DELETE',
          headers: {
            'Content-Type': 'application/json',
          },
        });
        
        if (!response.ok) {
          const errorData = await response.json().catch(() => ({}));
          throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
        }
        
        // Remove the item from the local array
        const index = this.varieties.findIndex(v => 
          v['nazwa odmiany'] === this.itemToDelete['nazwa odmiany']
        );
        if (index > -1) {
          this.varieties.splice(index, 1);
        }
        
        // Close dialog and reset
        this.showDeleteDialog = false;
        this.itemToDelete = null;
        
        console.log('Successfully deleted variety:', varietyName);
        
      } catch (error) {
        console.error('Error deleting variety:', error);
        this.error = `Błąd podczas usuwania odmiany: ${error.message}`;
        
        // Don't remove locally if API call fails - let user know about the error
        this.showDeleteDialog = false;
        this.itemToDelete = null;
      } finally {
        this.deleting = false;
      }
    },
    
    async generateManyVarieties() {
      if (!this.bulkCount || this.bulkCount < 1) {
        this.error = 'Wprowadź prawidłową liczbę odmian do wygenerowania (minimum 1).';
        return;
      }
      
      this.generatingMany = true;
      this.error = null;
      
      try {
        const response = await fetch(this.generateManyApiUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            count: this.bulkCount
          })
        });
        
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
        }
        
        const result = await response.json();
        console.log('Successfully generated varieties:', result);
        
        // Refresh the table to show the new varieties
        await this.fetchVarieties();
        
        // Reset bulk count
        this.bulkCount = 5;
        
      } catch (error) {
        console.error('Error generating many varieties:', error);
        this.error = `Błąd podczas generowania odmian: ${error.message}`;
      } finally {
        this.generatingMany = false;
      }
    },
    
async generateVariety() {
  if (!this.newVariety['nazwa odmiany'] || !this.newVariety.uwagi) {
    this.error = 'Wprowadź nazwę odmiany i uwagi aby wygenerować charakterystykę AI.';
    return;
  }
  
  this.generating = true;
  this.error = null;
  
  try {
    // Send the data with the exact field names the API expects
    const response = await fetch(this.generateApiUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        'nazwa odmiany': this.newVariety['nazwa odmiany'], // Changed from 'nazwa'
        uwagi: this.newVariety.uwagi
      })
    });
    
    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
    }
    
    const generatedVariety = await response.json();
    
    // Update the form with generated values
    this.newVariety['gęstość nasadzenia'] = generatedVariety['gęstość nasadzenia'] || '';
    this.newVariety['okres wegetacji'] = generatedVariety['okres wegetacji'] || '';
    this.newVariety['odporność na pękanie'] = generatedVariety['odporność na pękanie'] || 1;
    this.newVariety.masa = generatedVariety.masa || '';
    this.newVariety['przetwórstwo'] = generatedVariety['przetwórstwo'] || false;
    this.newVariety.przechowywanie = generatedVariety.przechowywanie || false;
    this.newVariety['świeży rynek'] = generatedVariety['świeży rynek'] || false;
    this.newVariety.generowane = true;
    
    // Refresh the table to show the new variety
    await this.fetchVarieties();
    
    // Clear the form after successful generation
    this.clearForm();
    
  } catch (error) {
    console.error('Error generating variety:', error);
    this.error = `Błąd podczas generowania odmiany: ${error.message}`;
  } finally {
    this.generating = false;
  }
},
    
    async addVariety() {
      if (!this.newVariety['nazwa odmiany']) return;
      
      this.submitting = true;
      this.error = null;
      
      try {
        const newEntry = {
          nazwa: this.newVariety['nazwa odmiany'],
          masa: this.newVariety.masa,
          'gęstość nasadzenia': this.newVariety['gęstość nasadzenia'],
          uwagi: this.newVariety.uwagi || 'Brak uwag',
          'okres wegetacji': this.newVariety['okres wegetacji'],
          'odporność na pękanie': this.newVariety['odporność na pękanie'],
          'przetwórstwo': this.newVariety['przetwórstwo'],
          przechowywanie: this.newVariety.przechowywanie,
          'świeży rynek': this.newVariety['świeży rynek']
        };
        
        // Use the dedicated /add endpoint for full mode
        const response = await fetch(this.addApiUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(newEntry)
        });
        
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
        }
        
        const addedVariety = await response.json();
        console.log('Successfully added variety:', addedVariety);
        
        // Refresh the table to show the new variety
        await this.fetchVarieties();
        
        // Clear the form after successful addition
        this.clearForm();
        
      } catch (error) {
        console.error('Error adding variety:', error);
        this.error = `Błąd podczas dodawania odmiany: ${error.message}`;
        
        // Fallback: add locally if API call fails
        const fallbackEntry = {
          'nazwa odmiany': this.newVariety['nazwa odmiany'],
          masa: this.newVariety.masa,
          'gęstość nasadzenia': this.newVariety['gęstość nasadzenia'],
          'okres wegetacji': this.newVariety['okres wegetacji'],
          'odporność na pękanie': this.newVariety['odporność na pękanie'],
          'przetwórstwo': this.newVariety['przetwórstwo'],
          przechowywanie: this.newVariety.przechowywanie,
          'świeży rynek': this.newVariety['świeży rynek'],
          uwagi: this.newVariety.uwagi || 'Brak uwag',
          generowane: false
        };
        this.varieties.push(fallbackEntry);
        this.clearForm();
      } finally {
        this.submitting = false;
      }
    },
    
    clearForm() {
      this.newVariety = {
        'nazwa odmiany': '',
        'gęstość nasadzenia': '',
        'okres wegetacji': '',
        'odporność na pękanie': 1,
        masa: '',
        'przetwórstwo': false,
        przechowywanie: false,
        'świeży rynek': false,
        uwagi: '',
        generowane: false
      };
    }
  }
};
</script>
  
  <style scoped>
/* General table styling */
:deep(.v-data-table thead th),
:deep(.v-data-table tbody td) {
  text-align: center !important;
  justify-content: center !important;
  border: 2px solid rgb(182, 173, 173);
  color: black;
}

:deep(.v-data-table tbody td) {
  background-color: rgb(255, 255, 255);
}

:deep(.v-data-table thead) {
  background-color: #9EB99D !important;
}

:deep(.v-data-table thead th) {
  color: #000000;
  border: 2px solid white !important;
  font-size: 1.1em;
}

/* Fixed "Odmiana" header styling - simplified approach */
:deep(.v-data-table thead th.odmiana-header-custom),
:deep(.v-data-table thead th.odmiana-header-custom .v-data-table-header__content),
:deep(.v-data-table thead th.odmiana-header-custom span) {
  background-color: rgb(9, 73, 7) !important;
  color: white !important;
  font-size: 1.2em !important;
  font-weight: bold !important;
  text-transform: uppercase !important;
  border: 2px solid white !important;
}

/* Ensure the header content wrapper displays properly */
:deep(.v-data-table thead th.odmiana-header-custom .v-data-table-header__content) {
  display: flex !important;
  align-items: center !important;
  justify-content: center !important;
  width: 100% !important;
  height: 100% !important;
}

/* Target any nested elements that might contain the text */
:deep(.v-data-table thead th.odmiana-header-custom *) {
  color: white !important;
}

/* Ensure header content is visible */
:deep(.v-data-table thead th .v-data-table-header__content) {
  display: flex;
  align-items: center;
  justify-content: center;
  color: inherit !important;
}

/* First column data cells */
:deep(.v-data-table tbody td:first-child) {
  background-color: rgb(223, 230, 213);
  color: rgb(7, 63, 5);
  font-size: 1.2em;
  font-weight: bold;
}

/* Symbol styling */
.przeznaczenie-symbols {
  color: rgb(240, 157, 33);
  font-weight: bold;
  font-size: 2.3em;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
}

.odpornosc-symbols {
  color: rgb(10, 78, 8);
  font-weight: bold;
  font-size: 2.3em;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
}

/* Data Entry Table */
.data-entry-table {
  width: 100%;
  border-collapse: collapse;
}

.data-entry-table th,
.data-entry-table td {
  border: 2px solid rgb(182, 173, 173);
  text-align: center;
  vertical-align: middle;
}

.entry-header {
  background-color: #9EB99D !important;
  color: #000000;
  font-size: 1.1em;
  padding: 12px 8px;
  font-weight: bold;
}

.entry-header.odmiana-header-custom {
  background-color: rgb(9, 73, 7) !important;
  color: white !important;
  font-size: 1.2em;
  text-transform: uppercase;
}

.entry-cell {
  background-color: rgb(255, 255, 255);
  padding: 8px;
  height: 60px;
}

.entry-cell.first-column {
  background-color: rgb(223, 230, 213);
  color: rgb(7, 63, 5);
  font-weight: bold;
}

.entry-cell.first-column .entry-input :deep(.v-field__input) {
  color: rgb(7, 63, 5) !important;
}

.entry-input {
  width: 100%;
}

.entry-input :deep(.v-field) {
  font-size: 0.9em;
}

.entry-input :deep(.v-field__input) {
  min-height: 30px;
  padding-top: 4px;
  padding-bottom: 4px;
  text-align: center;
}

.custom-card-title {
  font-size: 1.5rem;
  color: black;
  background-color: white;
  padding: 16px;
}

.table-wrapper {
  position: relative;
  overflow-x: auto;
}
  </style>