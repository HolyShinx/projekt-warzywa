<template>
  <v-container>
    <v-card>
      <v-card-title class="custom-card-title">
        Cebula - Charakterystyka odmian
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
      
      <v-data-table
        :headers="headers"
        :items="varieties"
        item-key="nazwa odmiany"
        class="elevation-1"
        :items-per-page="-1" 
        hide-default-footer
        :loading="loading"
      >
        <template v-slot:item.składowanie="{ item }">
          <span class="przechowanie-symbols">{{ getStorageSymbols(item.składowanie) }}</span>
        </template>
        
        <template v-slot:item.nazwa_odmiany="{ item }">
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
            <td class="text-center">{{ item.typ }}</td>
            <td class="text-center">{{ item.wczesność }}</td>
            <td class="text-center">{{ item.kolor }}</td>
            <td class="text-center">
              <span class="przechowanie-symbols">{{ getStorageSymbols(item.składowanie) }}</span>
            </td>
            <td class="text-center">{{ item.uwagi }}</td>
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
        Dodaj nową odmianę cebuli
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
          <strong>Tryb AI:</strong> Wprowadź nazwę odmiany i uwagi, a reszta będzie wygenerowana automatycznie.
        </v-alert>
      </div>

      <!-- Full Mode (Manual Entry) -->
      <div v-else class="table-wrapper">
        <v-table class="data-entry-table">
          <thead>
            <tr>
              <th class="odmiana-header-custom entry-header">Nazwa odmiany</th>
              <th class="entry-header">Typ</th>
              <th class="entry-header">Wczesność</th>
              <th class="entry-header">Kolor</th>
              <th class="entry-header">Składowanie</th>
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
                  v-model="newVariety.wczesność"
                  :items="wczenoscOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.kolor"
                  :items="kolorOptions"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="entry-input"
                />
              </td>
              <td class="entry-cell">
                <v-select
                  v-model="newVariety.składowanie"
                  :items="skladowanieOptions"
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
      apiUrl: 'http://192.168.1.54:3000/onion',
      addApiUrl: 'http://192.168.1.54:3000/onion/add',
      generateApiUrl: 'http://192.168.1.54:3000/onion/generate',
      generateManyApiUrl: 'http://192.168.1.54:3000/onion/generate-many',
      deleteApiUrl: 'http://192.168.1.54:3000/onion/by-name/VarietyName/full',
      headers: [
        {
          title: 'Nazwa odmiany',
          align: 'start',
          sortable: true,
          key: 'nazwa_odmiany',
          fixed: true,
          width: '250px',
          class: 'odmiana-header-custom',
        },
        { title: 'Typ', key: 'typ', sortable: true, width: '150px' },
        { title: 'Wczesność', key: 'wczesność', sortable: true, width: '180px' },
        { title: 'Kolor', key: 'kolor', sortable: true, width: '150px' },
        { title: 'Składowanie', key: 'składowanie', sortable: false, width: '120px' },
        { title: 'Uwagi', key: 'uwagi', sortable: false, width: '300px' },
      ],  
      varieties: [],
      newVariety: {
        'nazwa odmiany': '',
        typ: '',
        wczesność: '',
        kolor: '',
        składowanie: 1,
        uwagi: '',
        generowane: false
      },
      typOptions: ['Hiszpański', 'Rijnsburger'],
      wczenoscOptions: ['Bardzo wczesna', 'Wczesna', 'Średnio wczesna', 'Średnio późna', 'Późna'],
      kolorOptions: ['Brązowa', 'Czerwona', 'Jasnobrązowa', 'Ciemno żółtobrązowa', 'Żółta', 'Żółtobrązowa', 'Biała', 'Fioletowa'],
      skladowanieOptions: [
        { title: '+ (1)', value: 1 },
        { title: '++ (2)', value: 2 },
        { title: '+++ (3)', value: 3 },
        { title: '++++ (4)', value: 4 }
      ]
    };
  },
  computed: {
    isFullDataAvailable() {
      return this.newVariety['nazwa odmiany'] && 
             this.newVariety.typ && 
             this.newVariety.wczesność && 
             this.newVariety.kolor && 
             this.newVariety.składowanie && 
             this.newVariety.uwagi;
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
        this.varieties = Array.isArray(data) ? data : [];
        
        // Update dropdown options based on fetched data
        this.updateDropdownOptions();
        
      } catch (error) {
        console.error('Error fetching varieties:', error);
        this.error = `Błąd podczas pobierania danych: ${error.message}`;
        
        // Fallback to empty array if API fails
        this.varieties = [];
      } finally {
        this.loading = false;
      }
    },
    
    updateDropdownOptions() {
      // Extract unique values from fetched data to update dropdown options
      const types = [...new Set(this.varieties.map(v => v.typ).filter(Boolean))];
      const earliness = [...new Set(this.varieties.map(v => v.wczesność).filter(Boolean))];
      const colors = [...new Set(this.varieties.map(v => v.kolor).filter(Boolean))];
      
      // Merge with existing options to avoid losing predefined ones
      this.typOptions = [...new Set([...this.typOptions, ...types])];
      this.wczenoscOptions = [...new Set([...this.wczenoscOptions, ...earliness])];
      this.kolorOptions = [...new Set([...this.kolorOptions, ...colors])];
    },
    
    getStorageSymbols(storageLevel) {
      const symbols = {
        1: '+',
        2: '++',
        3: '+++',
        4: '++++'
      };
      return symbols[storageLevel] || '+';
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
        const response = await fetch(this.generateApiUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            nazwa: this.newVariety['nazwa odmiany'],
            uwagi: this.newVariety.uwagi
          })
        });
        
        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
        }
        
        const generatedVariety = await response.json();
        
        // Update the form with generated values
        this.newVariety.typ = generatedVariety.typ || '';
        this.newVariety.wczesność = generatedVariety.wczesnosc || generatedVariety.wczesność || '';
        this.newVariety.kolor = generatedVariety.kolor || '';
        this.newVariety.składowanie = generatedVariety.skladowanie || generatedVariety.składowanie || 1;
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
          typ: this.newVariety.typ || 'Nie określono',
          wczesnosc: this.newVariety.wczesność || 'Nie określono',
          kolor: this.newVariety.kolor || 'Nie określono',
          skladowanie: this.newVariety.składowanie || 1,
          uwagi: this.newVariety.uwagi || 'Brak uwag'
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
          typ: this.newVariety.typ || 'Nie określono',
          wczesność: this.newVariety.wczesność || 'Nie określono',
          kolor: this.newVariety.kolor || 'Nie określono',
          składowanie: this.newVariety.składowanie || 1,
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
        typ: '',
        wczesność: '',
        kolor: '',
        składowanie: 1,
        uwagi: '',
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
  background-color: rgb(221, 182, 96) !important; /* Force a solid header background */
}

:deep(.v-data-table thead th) {
  color: #000000;
  border: 2px solid white !important;
  font-size: 1.1em;
}

/* Ensure 'Nazwa odmiany' (first column) is styled differently */
:deep(.v-data-table thead th:first-child) {
  background-color: #775009 !important; /* Dark brown */
  color: white !important; /* White text */
  font-size: 1.2em !important; /* Slightly larger */
  font-weight: bold !important; /* Bolder */
  text-transform: uppercase !important; /* Optional: makes text stand out */
  text-align: center !important;
}

:deep(.v-data-table tbody td:first-child) {
  background-color:rgb(216, 213, 209);
  color: #775009;
  font-size: 1.2em;
  text-align: left !important;
  font-weight: bold !important; /* Bolder */
}

/* Special styling for generated varieties */
.generated-variety {
  color: #1976d2 !important; /* Blue color for generated varieties */
  font-style: italic;
  text-decoration: underline;
  text-decoration-style: dotted;
}

.normal-variety {
  color: #775009; /* Normal brown color */
}

.generated-icon {
  color: #1976d2 !important;
  vertical-align: middle;
}

/* Delete mode styling */
.deletable-row {
  cursor: pointer !important;
  transition: all 0.2s ease;
}

.deletable-row:hover {
  background-color: #ffebee !important;
  transform: scale(1.02);
}

.deletable-table-row {
  cursor: pointer !important;
  transition: all 0.2s ease;
}

.deletable-table-row:hover {
  background-color: #ffebee !important;
  transform: scale(1.01);
}

.deletable-table-row:hover td {
  background-color: #ffebee !important;
}

.delete-icon {
  color: #d32f2f !important;
  vertical-align: middle;
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

/* Style for symbols in 'Składowanie' column data cells: amber font */
.przechowanie-symbols {
  color: rgb(221, 182, 96);
  font-weight: bold; /* Optional: make symbols bolder */
  font-size: 1.2em;
}

.custom-card-title {
  font-size: 1.5rem; /* Adjust as needed */
  color: black;
  background-color: white;
  padding: 16px; /* Optional: adjust padding */
}

/* Mode Toggle Styling */
.mode-toggle {
  margin-bottom: 16px;
}

.mode-toggle .v-btn {
  font-weight: bold;
  text-transform: none;
}

/* Data Entry Table Styling */
.table-wrapper {
  position: relative;
  overflow-x: auto;
}

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
  background-color: rgb(221, 182, 96) !important;
  color: #000000;
  font-size: 1.1em;
  padding: 12px 8px;
  font-weight: bold;
}

.entry-header.odmiana-header-custom {
  background-color: #775009 !important;
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
  background-color: rgb(216, 213, 209);
}

.entry-cell.first-column .entry-input :deep(.v-field__field) {
  color: #775009;
}

.entry-cell.first-column .entry-input :deep(.v-field__input) {
  color: #775009 !important;
  font-weight: bold;
}

.entry-cell.first-column .entry-input :deep(.v-field__outline) {
  border-color: #775009 !important;
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

/* Gap utility for flexbox spacing */
.gap-3 {
  gap: 12px;
}
</style>