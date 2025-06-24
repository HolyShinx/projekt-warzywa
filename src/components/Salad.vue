<template>
  <v-container>
    <v-card>
      <v-card-title class="custom-card-title">
        Sałata - Charakterystyka odmian
      </v-card-title>
      
      <!-- Loading indicator -->
      <v-progress-linear 
        v-if="loading" 
        indeterminate 
        color="success"
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
      
      <v-data-table
        :headers="headers"
        :items="varieties"
        item-key="odmiana"
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
              <span v-if="Array.isArray(item.immunities) && item.immunities.length > 0">
                {{ item.immunities.join(', ') }}
              </span>
              <span v-else>-</span>
            </td>
            <td class="text-center">{{ item.okresUprawy }}</td>
          </tr>
        </template>
      </v-data-table>
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
        <v-btn value="simple" color="success">
          <v-icon left>mdi-auto-fix</v-icon>
          Tryb Szybki
        </v-btn>
        <v-btn value="full" color="primary">
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
        Dodaj nową odmianę sałaty
      </v-card-title>
      
      <!-- Simple Mode (AI Generation) -->
      <div v-if="entryMode === 'simple'" class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Odmiana</th>
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
            </tr>
          </tbody>
        </v-table>
        
        <div class="mt-3 text-center">
          <v-btn
            color="success"
            @click="generateVariety"
            :disabled="!newVariety.odmiana || generating"
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
          <strong>Tryb AI:</strong> Wprowadź nazwę odmiany, a reszta będzie wygenerowana automatycznie.
        </v-alert>
      </div>

      <!-- Full Mode (Manual Entry) -->
      <div v-else class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Odmiana</th>
              <th class="entry-header">Typ</th>
              <th class="entry-header">Immunities</th>
              <th class="entry-header">Okres uprawy</th>
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
                  v-model="newVariety.immunities"
                  :items="immunitiesOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                  multiple
                  chips
                  closable-chips
                  placeholder="Wybierz immunities"
                />
              </td>
              <td class="entry-cell">
                <v-text-field
                  v-model="newVariety.okresUprawy"
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
  name: 'VarietiesTable',
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
      // API URLs - adjust these to match your lettuce API endpoints
      apiUrl: 'http://192.168.1.54:3000/lettuce',
      addApiUrl: 'http://192.168.1.54:3000/lettuce/add',
      generateApiUrl: 'http://192.168.1.54:3000/lettuce/generate',
      generateManyApiUrl: 'http://192.168.1.54:3000/lettuce/generate-many',
      deleteApiUrl: 'http://192.168.1.54:3000/lettuce/by-name/VarietyName',
      headers: [
        {
          title: 'Odmiana',
          align: 'start',
          sortable: true,
          key: 'odmiana',
          fixed: true,
          width: '200px',
          class: 'odmiana-header-custom',
        },
        { title: 'Typ', key: 'typ', sortable: true, width: '150px' },
        { title: 'Immunities', key: 'immunities', sortable: false, width: '250px' },
        { title: 'Okres uprawy', key: 'okresUprawy', sortable: true, width: '200px' },
      ],  
      varieties: [
        {
          odmiana: 'Aurelian',
          typ: 'lodowa',
          immunities: ['Bl:37-38', 'Bl:40EU', 'Nr:0', 'LMV:1'],
          okresUprawy: 'wiosna',
          generowane: false
        },
        {
          odmiana: 'Sample Variety',
          typ: 'masłowa',
          immunities: ['Bl:29-35', 'Nr:0'],
          okresUprawy: 'lato',
          generowane: false
        }
      ],
      newVariety: {
        odmiana: '',
        typ: '',
        immunities: [],
        immunitiesText: '',
        okresUprawy: '',
        generowane: false
      },
      typOptions: ['masłowa', 'lodowa', 'rzymska', 'liściowa', 'dębowa', 'batavia'],
      immunitiesOptions: [
        'Bl:29-40',
        'Nr:0',
        'TBSV',
        'Bl:29-39',
        'Fol:4',
        'Bl:29-35',
        'Bl:37-38',
        'Bl:40EU',
        'Fol:1',
        'LMV:1',
        'Bl:29',
        'Bl:32',
        'Bl:34',
        'Bl:36',
        'Bl:30-32',
        'Bl:37-39'
      ],
    };
  },
  computed: {
    isFullDataAvailable() {
      return this.newVariety.odmiana && 
             this.newVariety.typ && 
             this.newVariety.okresUprawy;
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
        
        // Transform the data to match our table structure
        this.varieties = Array.isArray(data) ? data.map(item => ({
          odmiana: item['nazwa odmiany'] || item.odmiana || '',
          typ: item.typ || '',
          immunities: item.immunities || item.odporności ? 
            (Array.isArray(item.immunities) ? item.immunities : 
             Array.isArray(item.odporności) ? item.odporności : 
             typeof item.immunities === 'string' ? item.immunities.split(',').map(s => s.trim()) :
             typeof item.odporności === 'string' ? item.odporności.split(',').map(s => s.trim()) : 
             []) : [],
          okresUprawy: item['okres uprawy'] || item.okresUprawy || item.uwagi || '',
          generowane: item.generowane || false
        })) : [];
        
        // Update dropdown options based on fetched data
        this.updateDropdownOptions();
        
      } catch (error) {
        console.error('Error fetching varieties:', error);
        this.error = `Błąd podczas pobierania danych: ${error.message}`;
        
        // Keep existing fallback data if API fails
      } finally {
        this.loading = false;
      }
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
        
        // Fallback: remove locally if it's a generated variety
        if (this.itemToDelete.generowane) {
          const index = this.varieties.findIndex(v => 
            v.odmiana === this.itemToDelete.odmiana
          );
          if (index > -1) {
            this.varieties.splice(index, 1);
          }
        }
        
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
        
        // Fallback: generate some dummy varieties locally
        const generatedVarieties = [];
        for (let i = 0; i < this.bulkCount; i++) {
          const letterVarietyTypes = ['masłowa', 'lodowa', 'rzymska', 'liściowa', 'dębowa', 'batavia'];
          const seasons = ['wiosna', 'lato', 'jesień', 'wiosna-lato'];
          const sampleImmunities = [
            ['Bl:29-35', 'Nr:0'],
            ['Bl:37-38', 'LMV:1'],
            ['Bl:40EU', 'Nr:0', 'LMV:1'],
            ['Bl:29-35', 'Bl:37-38']
          ];
          
          generatedVarieties.push({
            odmiana: `Generowana ${i + 1}`,
            typ: letterVarietyTypes[Math.floor(Math.random() * letterVarietyTypes.length)],
            immunities: sampleImmunities[Math.floor(Math.random() * sampleImmunities.length)],
            okresUprawy: seasons[Math.floor(Math.random() * seasons.length)],
            generowane: true
          });
        }
        
        this.varieties.push(...generatedVarieties);
        this.bulkCount = 5;
      } finally {
        this.generatingMany = false;
      }
    },
    
    async generateVariety() {
      if (!this.newVariety.odmiana) {
        this.error = 'Wprowadź nazwę odmiany aby wygenerować charakterystykę AI.';
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
            "nazwa odmiany": this.newVariety.odmiana,
          })
        });
        
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
        }
        
        const generatedVariety = await response.json();
        
        // Update the form with generated values, handling different possible response formats
        this.newVariety.typ = generatedVariety.typ || '';
        
        // Handle immunities field - check multiple possible field names
        let immunities = generatedVariety.immunities || 
                        generatedVariety.odporności || 
                        generatedVariety.IR || 
                        generatedVariety.HR || 
                        [];
        
        // Ensure immunities is an array
        if (typeof immunities === 'string') {
          immunities = immunities.split(',').map(item => item.trim()).filter(Boolean);
        } else if (!Array.isArray(immunities)) {
          immunities = [];
        }
        
        this.newVariety.immunities = immunities;
        this.newVariety.immunitiesText = immunities.join(', ');
        
        // Handle period field - check multiple possible field names
        this.newVariety.okresUprawy = generatedVariety['okres uprawy'] || 
                                     generatedVariety.okresUprawy || 
                                     generatedVariety.uwagi || 
                                     '';
        this.newVariety.generowane = true;
        
        // Refresh the table to show the new variety
        await this.fetchVarieties();
        
        // Clear the form after successful generation
        this.clearForm();
        
      } catch (error) {
        console.error('Error generating variety:', error);
        this.error = `Błąd podczas generowania odmiany: ${error.message}`;
        
        // Fallback: generate a dummy variety locally
        const letterVarietyTypes = ['masłowa', 'lodowa', 'rzymska', 'liściowa', 'dębowa', 'batavia'];
        const seasons = ['wiosna', 'lato', 'jesień', 'wiosna-lato'];
        const sampleImmunities = [
          ['Bl:29-35', 'Nr:0'],
          ['Bl:37-38', 'LMV:1'],
          ['Bl:40EU', 'Nr:0', 'LMV:1']
        ];
        
        const fallbackVariety = {
          odmiana: this.newVariety.odmiana,
          typ: letterVarietyTypes[Math.floor(Math.random() * letterVarietyTypes.length)],
          immunities: sampleImmunities[Math.floor(Math.random() * sampleImmunities.length)],
          okresUprawy: seasons[Math.floor(Math.random() * seasons.length)],
          generowane: true
        };
        
        this.varieties.push(fallbackVariety);
        this.clearForm();
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
          'nazwa odmiany': this.newVariety.odmiana,
          typ: this.newVariety.typ || 'nie określono',
          immunities: this.newVariety.immunities || [],
          'okres uprawy': this.newVariety.okresUprawy || 'nie określono',
          generowane: false
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
          odmiana: this.newVariety.odmiana,
          typ: this.newVariety.typ || 'nie określono',
          immunities: this.newVariety.immunities || [],
          okresUprawy: this.newVariety.okresUprawy || 'nie określono',
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
        immunities: [],
        immunitiesText: '',
        okresUprawy: '',
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
    border: 2px solid rgb(173, 182, 173); /* Adds a slight border to differentiate - greenish */
    color: black;
  
  }
  :deep(.v-data-table tbody td) {
    background-color: rgb(255, 255, 255);
  
  }
  :deep(.v-data-table thead) {
    background-color: rgb(150, 240, 150) !important; /* Light green header background */
  }
  :deep(.v-data-table thead th) {
    color: #000000;
    border: 2px solid white !important;
    font-size: 1.1em;
  }
  
  /* Ensure 'Odmiana' (first column) is styled differently */
  :deep(.v-data-table thead th:first-child) {
    background-color: #1B5E20 !important; /* Dark green */
    color: white !important; /* White text */
    font-size: 1.2em !important; /* Slightly larger */
    font-weight: bold !important; /* Bolder */
    text-transform: uppercase !important; /* Optional: makes text stand out */
    text-align: center !important;
  }
  :deep(.v-data-table tbody td:first-child) {
    background-color: rgb(213, 216, 209);
    color: #2E7D32; /* Dark green text */
    font-size: 1.1em;
  }
  
  
  
  /* Ensure header content (text, sort icon) is vertically aligned */
  :deep(.v-data-table thead th .v-data-table-header__content) {
      display: flex;
      align-items: center;
      justify-content: center; /* This will center the text and icon if they are flex items */
      border: none !important; 
  }
  :deep(.v-data-table thead th:nth-child(-n+4)) {
    padding-left: 35px;
  }
  
  /* Style for 'Odmiana' column data cells: green background and white font */
  .odmiana-cell {
    background-color: #4CAF50; /* Green */
    color: white;
    padding: 8px 4px; /* Adjust padding as needed */
    height: 100%; /* Optional: ensures div fills cell height */
    display: flex;
    align-items: center;
    justify-content: center;
    box-sizing: border-box;
  }
  
  .custom-card-title {
    font-size: 1.5rem; /* Adjust as needed */
    color: black;
    background-color: white;
    padding: 16px; /* Optional: adjust padding */
  }
  .data-entry-table {
  border-collapse: separate;
  border-spacing: 0;
}

.data-entry-table thead th {
  background-color: rgb(150, 240, 150) !important;
  color: #000000 !important;
  border: 2px solid white !important;
  font-size: 1.1em !important;
  text-align: center !important;
  padding: 12px 8px !important;
}

.data-entry-table thead th:first-child {
  background-color: #1B5E20 !important;
  color: white !important;
  font-size: 1.2em !important;
  font-weight: bold !important;
  text-transform: uppercase !important;
}

.data-entry-table tbody td {
  background-color: rgb(255, 255, 255) !important;
  border: 2px solid rgb(173, 182, 173) !important;
  text-align: center !important;
  padding: 8px 4px !important;
}

.data-entry-table tbody td:first-child {
  background-color: rgb(213, 216, 209) !important;
  color: #2E7D32 !important;
  font-size: 1.1em !important;
}

/* Entry input fields styling */
.entry-input {
  width: 100%;
}

.entry-input :deep(.v-field__field) {
  background-color: white;
}

.entry-input :deep(.v-field--focused .v-field__field) {
  background-color: #f8f9fa;
}

/* Table wrapper for consistent spacing */
.table-wrapper {
  padding: 16px;
}

/* Entry cell styling */
.entry-cell {
  vertical-align: middle;
  min-height: 60px;
}

.entry-header {
  font-weight: 600;
  letter-spacing: 0.5px;
}
  </style>