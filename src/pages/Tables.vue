<template>
  <v-container>
    <v-select
      v-model="selectedTable"
      :items="tableOptions"
      label="Wybierz tabelę"
      class="mb-4"
    />
    <component :is="currentTableComponent" />
  </v-container>
</template>

<script>
import Califlour from '../components/Califlour.vue'
import Onion from '../components/Onion.vue'
import Tomato from '../components/Tomato.vue'
import Cabbage from '../components/Cabbage.vue'
import Salad from '../components/Salad.vue'

export default {
  name: 'DynamicTablesPage',
  components: {
    Califlour,
    Onion,
    Tomato,
    Cabbage,
    Salad,
  },
  data() {
    return {
      selectedTable: this.$route.params.table || 'Kalafior',
      tableOptions: ['Kalafior', 'Cebula', 'Pomidor', 'Kapusta Biała', 'Sałata'],
    };
  },
  computed: {
    currentTableComponent() {
      const map = {
        Kalafior: 'Califlour',
        Cebula: 'Onion',
        Pomidor: 'Tomato',
        'Kapusta Biała': 'Cabbage',
        Sałata: 'Salad',
      };
      return map[this.selectedTable] || 'Califlour';
    },
  },
  watch: {
    '$route.params.table'(newVal) {
      this.selectedTable = newVal;
    },
  },
};
</script>