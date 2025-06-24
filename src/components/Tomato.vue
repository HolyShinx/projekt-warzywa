<template>
  <v-container>
    <v-card>
      <v-card-title class="custom-card-title">
        Pomidor - Charakterystyka odmian
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
      
      <!-- Success message -->
      <v-alert
        v-if="successMessage"
        type="success"
        dismissible
        @click:close="successMessage = null"
        class="ma-4"
      >
        {{ successMessage }}
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
          item-key="nazwa_odmiany"
          class="elevation-1"
          :items-per-page="-1"
          hide-default-footer
          :loading="loading"
        >
          <!-- Make all table rows clickable in delete mode -->
          <template v-slot:item="{ item }">
            <tr 
              :class="{ 'deletable-table-row': entryMode === 'delete' }"
              @click="entryMode === 'delete' ? handleDeleteClick(item) : null"
            >
              <td class="entry-cell first-column">
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
              <td class="text-center">{{ item.wigor }}</td>
              <td class="text-center">{{ item['gęstość nasadzenia'] }}</td>
              <td class="text-center">{{ item.masa }} g</td>
              <td class="text-center">{{ getTransportResistance(item['wytrzymałość w transporcie']) }}</td>
              <td class="text-center">{{ getCrackingResistance(item['odporność na pękanie']) }}</td>
              <td class="text-center">{{ item['świeży rynek'] ? 'Tak' : 'Nie' }}</td>
              <td class="text-center">
                <v-chip
                  v-for="immunity in getHRImmunities(item)"
                  :key="immunity"
                  size="small"
                  color="success"
                  class="ma-1"
                >
                  {{ immunity }}
                </v-chip>
              </td>
              <td class="text-center">
                <v-chip
                  v-for="immunity in getIRImmunities(item)"
                  :key="immunity"
                  size="small"
                  color="info"
                  class="ma-1"
                >
                  {{ immunity }}
                </v-chip>
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
        Dodaj nową odmianę pomidora
      </v-card-title>
      
      <!-- Simple Mode (AI Generation) -->
      <div v-if="entryMode === 'simple'" class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Nazwa odmiany</th>
              <th class="entry-header">Uwagi</th>
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
                  placeholder="Wpisz nazwę odmiany"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety.uwagi"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                  placeholder="Wpisz opis/uwagi"
                />
              </td>
            </tr>
          </tbody>
        </v-table>
        
        <div class="mt-3 text-center">
          <v-btn
            color="success"
            @click="generateVariety"
            :disabled="!canGenerate || generating"
            :loading="generating"
            class="mr-2"
          >
            <v-icon left size="small">mdi-auto-fix</v-icon>
            Generuj z AI
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
          <strong>Tryb AI:</strong> Wprowadź nazwę odmiany i uwagi, a reszta będzie wygenerowana automatycznie przez sztuczną inteligencję.
        </v-alert>
      </div>

      <!-- Full Mode (Manual Entry) -->
      <div v-else class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Nazwa odmiany</th>
              <th class="entry-header">Wigor</th>
              <th class="entry-header">Gęstość nasadzenia</th>
              <th class="entry-header">Masa owocu (g)</th>
              <th class="entry-header">Wytrzymałość</th>
              <th class="entry-header">Odporność na pękanie</th>
              <th class="entry-header">Świeży rynek</th>
              <th class="entry-header">Odporność HR</th>
              <th class="entry-header">Odporność IR</th>
              <th class="entry-header">Uwagi</th>
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
                  placeholder="Nazwa odmiany"
                />
              </td>
              <td class="entry-cell">
                <v-number-input
                  v-model="newVariety.wigor"
                  :min="1"
                  :max="5"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-number-input
                  v-model="newVariety['gęstość nasadzenia']"
                  :min="1000"
                  :max="50000"
                  :step="1000"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-number-input
                  v-model="newVariety.masa"
                  :min="10"
                  :max="1000"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety['wytrzymałość w transporcie']"
                  :items="[1, 2, 3, 4, 5]"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety['odporność na pękanie']"
                  :items="[1, 2, 3, 4, 5]"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-switch
                  v-model="newVariety['świeży rynek']"
                  color="success"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-combobox
                  v-model="newVariety.immunitiesHR"
                  :items="hrImmunityOptions"
                  multiple
                  chips
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-combobox
                  v-model="newVariety.immunitiesIR"
                  :items="irImmunityOptions"
                  multiple
                  chips
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
                  placeholder="Uwagi"
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
  name: 'TomatoVarietiesTable',
  data() {
    return {
      loading: false,
      error: null,
      successMessage: null,
      submitting: false,
      generating: false,
      generatingMany: false,
      deleting: false,
      showGenerateInfo: true,
      showDeleteDialog: false,
      itemToDelete: null,
      entryMode: 'simple', // 'simple', 'full', or 'delete'
      bulkCount: 5, // Default bulk generation count
      
      // API URLs - Update these to match your tomato API endpoints
      apiUrl: 'http://192.168.1.54:3000/tomato',
      addApiUrl: 'http://192.168.1.54:3000/tomato/add',
      generateApiUrl: 'http://192.168.1.54:3000/tomato/generate',
      generateManyApiUrl: 'http://192.168.1.54:3000/tomato/generate-many',
      deleteApiUrl: 'http://192.168.1.54:3000/tomato/by-name/VarietyName',
      
      headers: [
        {
          title: 'Nazwa odmiany',
          align: 'start',
          sortable: true,
          key: 'nazwa odmiany',
          fixed: true,
          width: '200px',
          class: 'odmiana-header-custom',
        },
        { title: 'Wigor', key: 'wigor', sortable: true, width: '100px' },
        { title: 'Gęstość nasadzenia', key: 'gęstość nasadzenia', sortable: true, width: '150px' },
        { title: 'Masa owocu', key: 'masa', sortable: true, width: '120px' },
        { title: 'Wytrzymałość', key: 'wytrzymałość w transporcie', sortable: true, width: '130px' },
        { title: 'Odp. pękanie', key: 'odporność na pękanie', sortable: true, width: '130px' },
        { title: 'Świeży rynek', key: 'świeży rynek', sortable: true, width: '120px' },
        { title: 'Odporność HR', key: 'immunitiesHR', sortable: false, width: '150px' },
        { title: 'Odporność IR', key: 'immunitiesIR', sortable: false, width: '150px' },
        { title: 'Uwagi', key: 'uwagi', sortable: false, width: '300px' },
      ],
      
      varieties: [],
      
      newVariety: {
        'nazwa odmiany': '',
        wigor: 3,
        'gęstość nasadzenia': 21000,
        masa: 100,
        'wytrzymałość w transporcie': 3,
        'odporność na pękanie': 3,
        'świeży rynek': true,
        immunitiesHR: [],
        immunitiesIR: [],
        uwagi: '',
        generowane: false
      },
      
      // Immunity options for dropdowns
      hrImmunityOptions: ['S', 'Fol:0-1', 'Fol:0-2', 'Vd:0', 'Va:0', 'Vv', 'ToMV', 'Ty', 'TYLCV', 'TSWV'],
      irImmunityOptions: ['Pst', 'Mi', 'Ma', 'Mj', 'On', 'For', 'Pl', 'TSWV', 'FORL']
    };
  },
  
  computed: {
    canGenerate() {
      return this.newVariety['nazwa odmiany']?.trim() && this.newVariety.uwagi?.trim();
    },
    
    isFullDataAvailable() {
      return this.newVariety['nazwa odmiany']?.trim() && 
             this.newVariety.wigor && 
             this.newVariety['gęstość nasadzenia'] && 
             this.newVariety.masa && 
             this.newVariety['wytrzymałość w transporcie'] && 
             this.newVariety['odporność na pękanie'] && 
             this.newVariety.uwagi?.trim();
    }
  },
  
  async mounted() {
    await this.fetchVarieties();
  },
  
  methods: {
    async fetchVarieties() {
      this.loading = true;
      this.error = null;
      
      try {
        const response = await fetch(this.apiUrl);
        
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const data = await response.json();
        
        // Normalize the data to ensure consistent field names
        this.varieties = Array.isArray(data) ? data.map(variety => {
          return this.normalizeVarietyData(variety);
        }) : [];
        
        // Update dropdown options based on fetched data
        this.updateDropdownOptions();
        
      } catch (error) {
        console.error('Error fetching varieties:', error);
        this.error = `Błąd podczas pobierania danych: ${error.message}`;
        
        // Use fallback data from the JSON you provided
        this.varieties = this.getFallbackData().map(variety => this.normalizeVarietyData(variety));
      } finally {
        this.loading = false;
      }
    },

normalizeVarietyData(variety) {
  // Check if data is nested inside a 'tomato' object
  const actualData = variety.tomato || variety;
  
  console.log('Actual data to normalize:', actualData); // Debug log
  
  // Parse HR immunities from string format "HR: Fol:0-1, Va:0, Vd:0"
  let hrImmunities = [];
  if (actualData.HR && typeof actualData.HR === 'string' && actualData.HR.includes('HR:')) {
    hrImmunities = actualData.HR.replace('HR:', '').split(',').map(s => s.trim()).filter(s => s);
  } else if (actualData.immunitiesHR && Array.isArray(actualData.immunitiesHR)) {
    hrImmunities = actualData.immunitiesHR;
  } else if (variety.immunitiesHR && Array.isArray(variety.immunitiesHR)) {
    // Check if immunities are at the top level (from your API response)
    hrImmunities = variety.immunitiesHR;
  }
  
  // Parse IR immunities from string format "IR: Ma, Mi, Mj, Pst"
  let irImmunities = [];
  if (actualData.IR && typeof actualData.IR === 'string' && actualData.IR.includes('IR:')) {
    irImmunities = actualData.IR.replace('IR:', '').split(',').map(s => s.trim()).filter(s => s);
  } else if (actualData.immunitiesIR && Array.isArray(actualData.immunitiesIR)) {
    irImmunities = actualData.immunitiesIR;
  } else if (variety.immunitiesIR && Array.isArray(variety.immunitiesIR)) {
    // Check if immunities are at the top level (from your API response)
    irImmunities = variety.immunitiesIR;
  }
  
  const normalized = {
    // Convert string immunity data to arrays
    immunitiesHR: hrImmunities,
    immunitiesIR: irImmunities,
    // Use actualData (from tomato object) for the main fields
    'nazwa odmiany': actualData['nazwa odmiany'] || actualData.nazwa || '',
    wigor: actualData.wigor || 0,
    'gęstość nasadzenia': actualData['gęstość nasadzenia'] || 0,
    masa: actualData.masa || 0,
    'wytrzymałość w transporcie': actualData['wytrzymałość w transporcie'] || 0,
    'odporność na pękanie': actualData['odporność na pękanie'] || 0,
    'świeży rynek': actualData['świeży rynek'] !== undefined ? actualData['świeży rynek'] : true,
    uwagi: actualData.uwagi || '',
    generowane: variety.generowane || false
  };
  
  console.log('Final normalized variety:', normalized); // Debug log
  return normalized;
},

    getFallbackData() {
      // Return the JSON data you provided as fallback
      return [
        {"nazwa odmiany":"Miceno F1","wigor":3,"gęstość nasadzenia":25000,"uwagi":"odporność i system korzeniowy – odmiana na trudne warunki","masa":85,"generowane":false,"wytrzymałość w transporcie":3,"odporność na pękanie":3,"odporności":"Fol:0-1","świeży rynek":true,"rodzaj przeznaczenia":"Standardowe","ir":false,"hr":true,"IR":"IR: Ma, Mi, Mj, Pst","HR":"HR: Fol:0-1, Va:0, Vd:0, TSWV"},
        {"nazwa odmiany":"N Redix F1","wigor":4,"gęstość nasadzenia":25000,"uwagi":"dobre plonowanie i wysoki Brix – intensywny czerwony kolor","masa":95,"generowane":false,"wytrzymałość w transporcie":3,"odporność na pękanie":3,"odporności":"Fol:0-1","świeży rynek":true,"rodzaj przeznaczenia":"Standardowe","ir":false,"hr":true,"IR":"IR: Ma, Mi, Mj","HR":"HR: Fol:0-1, TSWV"},
        {"nazwa odmiany":"Brixtone F1","wigor":4,"gęstość nasadzenia":21000,"uwagi":"jednolite dojrzewanie – idealny do zbioru mechanicznego, wysoki plon","masa":105,"generowane":false,"wytrzymałość w transporcie":3,"odporność na pękanie":3,"odporności":"Fol:0-1","świeży rynek":true,"rodzaj przeznaczenia":"Standardowe","ir":false,"hr":true,"IR":"IR: Ma, Mi, Mj, Pst","HR":"HR: Fol:0-1"},
        {"nazwa odmiany":"Ulisse F1","wigor":3,"gęstość nasadzenia":25000,"uwagi":"duża trwałość i odporność na uszkodzenia mechaniczne","masa":95,"generowane":false,"wytrzymałość w transporcie":3,"odporność na pękanie":3,"odporności":"Fol:0-1","świeży rynek":true,"rodzaj przeznaczenia":"Standardowe","ir":false,"hr":true,"IR":"IR: Ma, Mi, Mj","HR":"HR: Fol:0-1"},
        {"nazwa odmiany":"Bobcat F1","wigor":3,"gęstość nasadzenia":17500,"uwagi":"duży potencjał plonowania owoców o wysokiej jakości","masa":250,"generowane":false,"wytrzymałość w transporcie":2,"odporność na pękanie":2,"odporności":"Fol:0-1","świeży rynek":true,"rodzaj przeznaczenia":"Standardowe","ir":false,"hr":true,"IR":"","HR":"HR: Fol:0-1, Va:0, Vd:0"},
        {"nazwa odmiany":"Hapynet F1","wigor":3,"gęstość nasadzenia":17500,"uwagi":"malinowy do uprawy w gruncie","masa":200,"generowane":false,"wytrzymałość w transporcie":2,"odporność na pękanie":2,"odporności":"Fol:0-1","świeży rynek":true,"rodzaj przeznaczenia":"Standardowe","ir":false,"hr":true,"IR":"","HR":"HR: Fol:0-1"}
      ];
    },

    getHRImmunities(item) {
      return item.immunitiesHR || [];
    },

    getIRImmunities(item) {
      return item.immunitiesIR || [];
    },

    updateDropdownOptions() {
      // Extract unique immunity values from fetched data
      const allHR = this.varieties.flatMap(v => v.immunitiesHR || []);
      const allIR = this.varieties.flatMap(v => v.immunitiesIR || []);
      
      this.hrImmunityOptions = [...new Set([...this.hrImmunityOptions, ...allHR])];
      this.irImmunityOptions = [...new Set([...this.irImmunityOptions, ...allIR])];
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
        const varietyName = this.itemToDelete['nazwa odmiany'];
        const deleteUrl = this.deleteApiUrl.replace('VarietyName', encodeURIComponent(varietyName));
        
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
        
        this.showDeleteDialog = false;
        this.itemToDelete = null;
        
      } catch (error) {
        console.error('Error deleting variety:', error);
        this.error = `Błąd podczas usuwania odmiany: ${error.message}`;
        
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
  if (!this.newVariety['nazwa odmiany']?.trim() || !this.newVariety.uwagi?.trim()) {
    this.error = 'Wprowadź nazwę odmiany i uwagi aby wygenerować charakterystykę AI.';
    return;
  }
  
  this.generating = true;
  this.error = null;
  
  try {
    const response = await fetch(this.generateApiUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        'nazwa odmiany': this.newVariety['nazwa odmiany'],
        uwagi: this.newVariety.uwagi
      })
    });
    
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
    }
    
    const generatedVariety = await response.json();
    console.log('Generated variety from API:', generatedVariety); // Debug log
    
    // Normalize the generated variety data
    const normalizedVariety = this.normalizeVarietyData({
      ...generatedVariety,
      generowane: true
    });
    
    console.log('Normalized variety:', normalizedVariety); // Debug log
    
    // Use Vue's reactivity to add the new variety
    this.varieties = [...this.varieties, normalizedVariety];
    
    // Wait for next tick before clearing form and showing success
    await this.$nextTick();
    
    // Clear the form after successful generation
    this.clearForm();
    
    this.successMessage = 'Odmiana została pomyślnie wygenerowana!';
    
    // Clear success message after 3 seconds
    setTimeout(() => {
      this.successMessage = null;
    }, 3000);
    
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
      
      // Store the variety data before making the API call
      const varietyToAdd = {
        ...this.newVariety,
        generowane: false
      };
      
      try {
        const response = await fetch(this.addApiUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(varietyToAdd)
        });
        
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
        }
        
        const addedVariety = await response.json();
        console.log('Successfully added variety:', addedVariety);
        
        // Clear the form first
        this.clearForm();
        
        // Refresh the table to show the new variety
        await this.fetchVarieties();
        
        this.successMessage = 'Odmiana została pomyślnie dodana!';
        
      } catch (error) {
        console.error('Error adding variety:', error);
        this.error = `Błąd podczas dodawania odmiany: ${error.message}`;
        
        // Fallback: add locally if API call fails - use the stored variety data
        this.varieties.push(this.normalizeVarietyData(varietyToAdd));
        this.clearForm();
      } finally {
        this.submitting = false;
      }
    },

    clearForm() {
      this.newVariety = {
        'nazwa odmiany': '',
        wigor: 3,
        'gęstość nasadzenia': 21000,
        masa: 100,
        'wytrzymałość w transporcie': 3,
        'odporność na pękanie': 3,
        'świeży rynek': true,
        immunitiesHR: [],
        immunitiesIR: [],
        uwagi: '',
        generowane: false
      };
    },

    getTransportResistance(value) {
      const labels = {
        1: 'Bardzo niska',
        2: 'Niska',
        3: 'Średnia',
        4: 'Wysoka',
        5: 'Bardzo wysoka'
      };
      return labels[value] || 'Nieznana';
    },

    getCrackingResistance(value) {
      const labels = {
        1: 'Bardzo niska',
        2: 'Niska',
        3: 'Średnia',
        4: 'Wysoka',
        5: 'Bardzo wysoka'
      };
      return labels[value] || 'Nieznana';
    }
  }
};
</script> 
<style scoped>
/* General styling for the table */

/* Center all text in header and data cells */
:deep(.v-data-table thead th), /* Target inner content for centering */
:deep(.v-data-table tbody td) {
  text-align: center !important;
  justify-content: center !important; /* For flex content within header */
  border: 2px solid rgb(182, 173, 173); /* Adds a slight border to differentiate */
  color: black;
}

:deep(.v-data-table tbody td) {
  background-color: rgb(255, 255, 255);
}

:deep(.v-data-table thead) {
  background-color: rgb(240, 150, 150) !important; /* Force a solid header background */
}

:deep(.v-data-table thead th) {
  color: #000000;
  border: 2px solid white !important;
  font-size: 1.1em;
  
}

/* Ensure 'Odmiana' (first column) is styled differently - RED BACKGROUND */
:deep(.v-data-table thead th.odmiana-header-custom) {
  color: white !important;
  font-size: 1.2em;
  font-weight: bold;
  text-transform: uppercase;
  text-align: center;
  border: none !important;

}

/* Ensure the entire cell—including padding—is filled */
:deep(.v-data-table thead th.odmiana-header-custom::before),
:deep(.v-data-table thead th.odmiana-header-custom::after) {
  content: "";
  display: block;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgb(148, 14, 14) !important;
  z-index: -1;
}
:deep(.v-data-table tbody td:first-child) {
  background-color: red; /* Changed to pure red */
  color: white; /* Changed to white text for better contrast on red background */
  font-size: 1.1em;
}

/* Ensure header content (text, sort icon) is vertically aligned */
:deep(.v-data-table thead th .v-data-table-header__content) {
    display: flex;
    align-items: center;
    justify-content: center; /* This will center the text and icon if they are flex items */
    border: none !important; 
    
}

:deep(.v-data-table thead th:nth-child(n+2):nth-child(-n+4)) {
  padding-left: 35px;
}
:deep(.v-data-table thead th:nth-child(n+6):nth-child(-n+7)) {
  padding-left: 35px;
}

.custom-card-title {
  font-size: 1.5rem; /* Adjust as needed */
  color: black;
  background-color: white;
  padding: 16px; /* Optional: adjust padding */
}

.table-wrapper {
  position: relative;
  overflow-x: auto;
}

/* Data Entry Table Styling */
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
  background-color: rgb(240, 150, 150) !important;
  color: #000000;
  font-size: 1.1em;
  padding: 12px 8px;
  font-weight: bold;
}

.entry-header.odmiana-header-custom {
  background-color: rgb(148, 14, 14) !important;
  color: white !important;
  font-size: 1.2em;
  text-transform: uppercase;
}

.owoc-header-span {
  border-bottom: none !important;
}

.entry-subheader {
  background-color: rgb(240, 150, 150) !important;
  color: #000000;
  font-size: 0.9em;
  padding: 8px 4px;
  font-weight: normal;
}

.subheader-row th:not(.entry-subheader) {
  border: none !important;
  background: transparent !important;
}

.entry-cell {
  background-color: rgb(255, 255, 255);
  padding: 8px;
  height: 60px;
}

.entry-cell.first-column {
  background-color: red;
}

.entry-cell.first-column .entry-input :deep(.v-field__field) {
  color: white;
}

.entry-cell.first-column .entry-input :deep(.v-field__input) {
  color: white !important;
}

.entry-cell.first-column .entry-input :deep(.v-field__outline) {
  border-color: white !important;
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
}
</style>