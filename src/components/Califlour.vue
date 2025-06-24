<template>
  <v-container>
    <v-card>
      <v-card-title class="custom-card-title">
        Kalafior - Charakterystyka odmian
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
          item-key="odmiana"
          class="elevation-1"
          :items-per-page="-1"
          hide-default-footer
          :loading="loading"
        >
          <!-- Custom slot for odmiana header -->
          <template v-slot:header.odmiana="{ column }">
            <th class="odmiana-header-custom">
              {{ column.text }}
            </th>
          </template>
          
          <!-- Symbol rendering templates -->
          <template v-slot:item.sila="{ item }">
            <span class="sila-symbols">{{ item.sila }}</span>
          </template>
          
          <template v-slot:item.odpornosc="{ item }">
            <span class="odpornosc-symbols">{{ item.odpornosc }}</span>
          </template>
          
          <template v-slot:item.okrywanie="{ item }">
            <span class="roza-symbols">{{ item.okrywanie }}</span>
          </template>
          
          <template v-slot:item.jakosc="{ item }">
            <span class="roza-symbols">{{ item.jakosc }}</span>
          </template>
          
          <template v-slot:item.swiezyRynek="{ item }">
            <span class="przeznaczenie-symbols">{{ item.swiezyRynek }}</span>
          </template>
          
          <template v-slot:item.zamrazanie="{ item }">
            <span class="przeznaczenie-symbols">{{ item.zamrazanie }}</span>
          </template>

          <!-- Variety name with delete functionality -->
          <template v-slot:item.odmiana="{ item }">
            <span 
              :class="{ 
                'generated-variety': item.generowane, 
                'normal-variety': !item.generowane,
                'deletable-row': entryMode === 'delete'
              }"
              @click="entryMode === 'delete' ? handleDeleteClick(item) : null"
            >
              {{ item.odmiana }}
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
                  {{ item.odmiana }}
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
              <td class="text-center">{{ item.typ }}</td>
              <td class="text-center">
                <span class="sila-symbols">{{ item.sila }}</span>
              </td>
              <td class="text-center">
                <span class="odpornosc-symbols">{{ item.odpornosc }}</span>
              </td>
              <td class="text-center">
                <span class="roza-symbols">{{ item.okrywanie }}</span>
              </td>
              <td class="text-center">
                <span class="roza-symbols">{{ item.jakosc }}</span>
              </td>
              <td class="text-center">
                <span class="przeznaczenie-symbols">{{ item.swiezyRynek }}</span>
              </td>
              <td class="text-center">
                <span class="przeznaczenie-symbols">{{ item.zamrazanie }}</span>
              </td>
              <td class="text-center">{{ item.typWczesnosci }}</td>
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
          Czy na pewno chcesz usunąć odmianę <strong>"{{ itemToDelete?.odmiana }}"</strong>?
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
        Dodaj nową odmianę kalafiora
      </v-card-title>
      
      <!-- Simple Mode (AI Generation) -->
      <div v-if="entryMode === 'simple'" class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Odmiana</th>
              <th class="entry-header">Cecha Charakterystyczna</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td class="entry-cell first-column">
                <v-text-field
                  v-model="newVariety.odmiana"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety.typWczesnosci"
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
            :disabled="!newVariety.odmiana || !newVariety.typWczesnosci || generating"
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
          <strong>Tryb AI:</strong> Wprowadź nazwę odmiany i cechę charakterystyczną, a reszta będzie wygenerowana automatycznie.
        </v-alert>
      </div>

      <!-- Full Mode (Manual Entry) -->
      <div v-else class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Odmiana</th>
              <th class="entry-header">Okres wegetacji</th>
              <th class="entry-header">Siła wzrostu</th>
              <th class="entry-header">Odporność</th>
              <th class="entry-header">Okrywanie</th>
              <th class="entry-header">Jakość</th>
              <th class="entry-header">Świeży rynek</th>
              <th class="entry-header">Zamrażanie</th>
              <th class="entry-header">Cecha Charakterystyczna</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td class="entry-cell first-column">
                <v-text-field
                  v-model="newVariety.odmiana"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.typ"
                  :items="typOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.sila"
                  :items="silaOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.odpornosc"
                  :items="odpornoscOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.okrywanie"
                  :items="okrywanieOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.jakosc"
                  :items="jakoscOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.swiezyRynek"
                  :items="zastosowanieOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.zamrazanie"
                  :items="zastosowanieOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety.typWczesnosci"
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
  name: 'CauliflowerVarietiesTable',
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
      
      // API URLs - adapt these to your cauliflower endpoints
      apiUrl: 'http://192.168.1.54:3000/cauliflower',
      addApiUrl: 'http://192.168.1.54:3000/cauliflower/add',
      generateApiUrl: 'http://192.168.1.54:3000/cauliflower/generate',
      generateManyApiUrl: 'http://192.168.1.54:3000/cauliflower/generate-many',
      deleteApiUrl: 'http://192.168.1.54:3000/cauliflower/by-name/VarietyName',
      
      headers: [
        {
          text: 'Odmiana',
          value: 'odmiana',
          align: 'start',
          sortable: true,
          fixed: true,
          width: '200px',
          class: 'odmiana-column',
        },
        { title: 'Okres wegetacji', key: 'typ', sortable: true, width: '150px' },
        { title: 'Siła wzrostu rośliny', key: 'sila', sortable: true, width: '150px' },
        { title: 'Odporność na warunki stresowe', key: 'odpornosc', sortable: true, width: '150px' },
        { 
          title: 'Róża', 
          key: 'owoc', 
          sortable: false, 
          width: '450px',
          class: 'owoc-header-custom',
          children: [
            { 
              title: 'okrywanie', 
              key: 'okrywanie', 
              sortable: true, 
              width: '150px',
              class: 'masa-column'
            },
            { 
              title: 'jakość', 
              key: 'jakosc', 
              sortable: true, 
              width: '150px',
              class: 'owoc-child-header' 
            },
          ]
        },
        { 
          title: 'Zastosowanie', 
          key: 'owoc', 
          sortable: false, 
          width: '450px',
          class: 'owoc-header-custom',
          children: [
            { 
              title: 'świeży rynek', 
              key: 'swiezyRynek', 
              sortable: true, 
              width: '150px',
              class: 'masa-column'
            },
            { 
              title: 'zamrażanie', 
              key: 'zamrazanie', 
              sortable: true, 
              width: '150px',
              class: 'owoc-child-header' 
            },
          ]
        },
        { title: 'Cecha Charakterystyczna', key: 'typWczesnosci', sortable: true, width: '180px' },
      ],
      
      varieties: [],
      
      newVariety: {
        odmiana: '',
        typ: '',
        sila: '',
        odpornosc: '',
        okrywanie: '',
        jakosc: '',
        swiezyRynek: '',
        zamrazanie: '',
        typWczesnosci: '',
        generowane: false
      },
      
      // Dropdown options
      typOptions: ['Wczesny', 'Średni', 'Późny'],
      silaOptions: ['■', '■■', '■■■'],
      odpornoscOptions: ['■', '■■', '■■■'],
      okrywanieOptions: ['■', '■■', '■■■'],
      jakoscOptions: ['■', '■■', '■■■'],
      zastosowanieOptions: ['+', '++', '+++']
    };
  },
  
  computed: {
    isFullDataAvailable() {
      return this.newVariety.odmiana && 
             this.newVariety.typ && 
             this.newVariety.sila && 
             this.newVariety.odpornosc && 
             this.newVariety.okrywanie && 
             this.newVariety.jakosc && 
             this.newVariety.swiezyRynek && 
             this.newVariety.zamrazanie && 
             this.newVariety.typWczesnosci;
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
      
      // Map the JSON field names to component field names
      this.varieties = Array.isArray(data) ? data.map(item => ({
        odmiana: item['nazwa odmiany'] || item.odmiana || '',
        typ: this.mapPeriodToType(item['okres wegetacji']) || item.typ || '',
        sila: this.mapNumberToSymbols(item['siła wzrostu']) || item.sila || '',
        odpornosc: this.mapNumberToSymbols(item['odporność na stres']) || item.odpornosc || '',
        okrywanie: this.mapNumberToSymbols(item['okrywanie']) || item.okrywanie || '',
        jakosc: this.mapNumberToSymbols(item['jakość']) || item.jakosc || '',
        swiezyRynek: this.mapBooleanToSymbols(item['świeży rynek']) || item.swiezyRynek || '',
        zamrazanie: this.mapBooleanToSymbols(item['zamrażanie']) || item.zamrazanie || '',
        typWczesnosci: item['uwagi'] || item.typWczesnosci || '',
        generowane: item['generowane'] || false
      })) : this.varieties;
      
      // Update dropdown options based on fetched data
      this.updateDropdownOptions();
      
    } catch (error) {
      console.error('Error fetching varieties:', error);
      this.error = `Błąd podczas pobierania danych: ${error.message}`;
    } finally {
      this.loading = false;
    }
  },

  // Helper method to map vegetation period to type
  mapPeriodToType(period) {
    if (!period) return '';
    const days = parseInt(period);
    if (days <= 70) return 'Wczesny';
    if (days <= 90) return 'Średni';
    return 'Późny';
  },

  // Helper method to map numbers (1-5) to symbols (■)
  mapNumberToSymbols(number) {
    if (!number) return '';
    const num = parseInt(number);
    if (num <= 2) return '■';
    if (num <= 4) return '■■';
    return '■■■';
  },

  // Helper method to map boolean to + symbols
  mapBooleanToSymbols(value) {
    if (value === true) return '+++';
    if (value === false) return '+';
    return '++'; // default for undefined/null
  },
    
    updateDropdownOptions() {
      // Extract unique values from fetched data to update dropdown options
      const types = [...new Set(this.varieties.map(v => v.typ).filter(Boolean))];
      
      // Merge with existing options to avoid losing predefined ones
      this.typOptions = [...new Set([...this.typOptions, ...types])];
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
        const varietyName = this.itemToDelete.odmiana;
        
        // Construct the URL by replacing the template parameter with the actual variety name
        const deleteUrl = this.deleteApiUrl.replace('VarietyName', encodeURIComponent(varietyName));
        
        console.log('Delete URL:', deleteUrl);
        console.log('Deleting variety:', varietyName);
        
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
          v.odmiana === this.itemToDelete.odmiana
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
        
        // Don't remove locally if API call fails
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
      if (!this.newVariety.odmiana || !this.newVariety.typWczesnosci) {
        this.error = 'Wprowadź nazwę odmiany i cechę charakterystyczną aby wygenerować charakterystykę AI.';
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
            nazwa: this.newVariety.odmiana,
            uwagi: this.newVariety.typWczesnosci
          })
        });
        
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
        }
        
        const generatedVariety = await response.json();
        
        // Update the form with generated values
        this.newVariety.typ = generatedVariety.typ || '';
        this.newVariety.sila = generatedVariety.sila || '';
        this.newVariety.odpornosc = generatedVariety.odpornosc || '';
        this.newVariety.okrywanie = generatedVariety.okrywanie || '';
        this.newVariety.jakosc = generatedVariety.jakosc || '';
        this.newVariety.swiezyRynek = generatedVariety.swiezyRynek || '';
        this.newVariety.zamrazanie = generatedVariety.zamrazanie || '';
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
      if (!this.newVariety.odmiana) return;
      
      this.submitting = true;
      this.error = null;
      
      try {
        const newEntry = {
          odmiana: this.newVariety.odmiana,
          typ: this.newVariety.typ || 'Nie określono',
          sila: this.newVariety.sila || 'X',
          odpornosc: this.newVariety.odpornosc || 'X',
          okrywanie: this.newVariety.okrywanie || 'X',
          jakosc: this.newVariety.jakosc || 'X',
          swiezyRynek: this.newVariety.swiezyRynek || 'X',
          zamrazanie: this.newVariety.zamrazanie || 'X',
          typWczesnosci: this.newVariety.typWczesnosci || 'Nie określono',
          generowane: this.newVariety.generowane || false
        };

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
          odmiana: this.newVariety.odmiana,
          typ: this.newVariety.typ || 'Nie określono',
          sila: this.newVariety.sila || 'X',
          odpornosc: this.newVariety.odpornosc || 'X',
          okrywanie: this.newVariety.okrywanie || 'X',
          jakosc: this.newVariety.jakosc || 'X',
          swiezyRynek: this.newVariety.swiezyRynek || 'X',
          zamrazanie: this.newVariety.zamrazanie || 'X',
          typWczesnosci: this.newVariety.typWczesnosci || 'Nie określono',
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
        odmiana: '',
        typ: '',
        sila: '',
        odpornosc: '',
        okrywanie: '',
        jakosc: '',
        swiezyRynek: '',
        zamrazanie: '',
        typWczesnosci: '',
        generowane: false
      };
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
    background-color: rgb(194, 221, 165) !important; /* Force a solid header background */
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
    background-color: rgb(105, 180, 19); /* Changed to white text for better contrast on red background */
    z-index: -1;
  }
  :deep(.v-data-table tbody td:first-child) {
    background-color: rgb(223, 230, 213); /* Changed to pure red */
    color: rgb(105, 180, 19); /* Changed to white text for better contrast on red background */
    font-size: 1.2em;
    font-weight: bold;
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
  .przeznaczenie-symbols {
    color: rgb(26, 172, 46);
    font-weight: bold;
    font-size: 2.3em; /* Increased from 2em to 2.5em */
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100%;
  }
  
  .odpornosc-symbols {
    color: rgb(240, 157, 33);
    font-weight: bold;
    font-size: 2.3em; /* Increased from 2em to 2.5em */
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100%;
  }

  .sila-symbols {
    color: rgb(97, 206, 25);
    font-weight: bold;
    font-size: 2.3em; /* Increased from 2em to 2.5em */
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100%;
  }

  .roza-symbols {
    color: rgb(98, 100, 97);
    font-weight: bold;
    font-size: 2.3em; /* Increased from 2em to 2.5em */
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100%;
  }
  
  /* Apply specific styling to the cell containing przeznaczenie symbols */
  :deep(.v-data-table tbody td:has(.odpornosc-symbols)) {
    padding: 0; /* Remove default cell padding */
    height: auto; /* Let the content determine the height */
    vertical-align: middle;
  }
  :deep(.data-entry-table thead th),
:deep(.data-entry-table tbody td) {
  text-align: center !important;
  justify-content: center !important;
  border: 2px solid rgb(182, 173, 173);
  color: black;
}

:deep(.data-entry-table tbody td) {
  background-color: rgb(255, 255, 255);
}

:deep(.data-entry-table thead) {
  background-color: rgb(194, 221, 165) !important;
}

:deep(.data-entry-table thead th) {
  color: #000000;
  border: 2px solid white !important;
  font-size: 1.1em;
}

/* Style the first column (Odmiana) in data entry tables */
:deep(.data-entry-table thead th.odmiana-header-custom) {
  color: white !important;
  font-size: 1.2em;
  font-weight: bold;
  text-transform: uppercase;
  text-align: center;
  border: none !important;
  background-color: rgb(105, 180, 19);
}

:deep(.data-entry-table tbody td.first-column) {
  color: rgb(105, 180, 19);
  font-size: 1.2em;
  font-weight: bold;
}

/* Entry input styling */
.entry-input {
  background-color: white;
}

.entry-input :deep(.v-field__input) {
  text-align: center;
  font-weight: bold;
}

.entry-input :deep(.v-select__selection) {
  text-align: center;
  font-weight: bold;
}
  </style>