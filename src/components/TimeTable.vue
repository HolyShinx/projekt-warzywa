<template>
  <div class="table-container">
    <v-data-table
      :headers="headers"
      :items="Odmiana"
      item-key="odmiana"
      class="elevation-1"
      density="comfortable"
    >
      <!-- First column header -->
      <template v-slot:header.odmiana>
        <th style="width:150px" scope="col">Odmiana</th>
      </template>

      <!-- Month headers -->
      <template
        v-for="m in months"
        :key="m.value"
        v-slot:[`header.${m.value}`]
      >
        <th style="width:100px" scope="col">{{ m.text }}</th>
      </template>

      <!-- Single <td colspan> with CSS‑gradient background -->
      <template #item="{ item }">
        <tr>
          <td><strong>{{ item.odmiana }}</strong></td>
          <td :colspan="months.length" class="timeline-cell">
            <div
              v-for="(seg, idx) in item.segments"
              :key="idx"
              class="bar"
              :class="seg.type === 'green' ? 'blue-bar' : 'orange-bar'"
              :style="getBarStyle(seg.start, seg.end)"
            ></div>
          </td>
        </tr>
      </template>
    </v-data-table>
  </div>
</template>

<script>
export default {
  data() {
    return {
      months: [
        { text: 'Styczeń', value: '01' }, { text: 'Luty', value: '02' },
        { text: 'Marzec', value: '03' }, { text: 'Kwiecień', value: '04' },
        { text: 'Maj', value: '05' }, { text: 'Czerwiec', value: '06' },
        { text: 'Lipiec', value: '07' }, { text: 'Sierpień', value: '08' },
        { text: 'Wrzesień', value: '09' }, { text: 'Październik', value: '10' },
        { text: 'Listopad', value: '11' }, { text: 'Grudzień', value: '12' },
      ],
      Odmiana: [
      {
    odmiana: 'Speedstar F1',
    segments: [
      { type: 'green', start: '2025-02-26', end: '2025-03-20' },
      { type: 'orange', start: '2025-04-28', end: '2025-06-02' },
    ]
  },
        { odmiana: 'EasyTop F1',
          segments:  [
          { type: 'green', start: '2025-02-26', end: '2025-03-20' },
          { type: 'orange', start: '2025-04-28', end: '2025-06-02' }]},
        { odmiana: 'Soilstar F1',
          segments: [
          { type: 'green', start: '2025-02-26', end: '2025-04-10'},
          { type: 'orange', start: '2025-05-03', end: '2025-06-12' }]},
        { odmiana: 'Bering F1',
          segments: [ 
          { type: 'green', start: '2025-03-13', end: '2025-04-12' },
          { type: 'orange', start: '2025-05-08', end: '2025-06-16'}]},
        { odmiana: 'Soilstar F1',
          segments: [
          { type: 'green', start: '2025-03-05', end: '2025-04-27'},
          { type: 'orange', start: '2025-05-22', end: '2025-07-08' },
          { type: 'green', start: '2025-06-05', end: '2025-07-27'},
          { type: 'orange', start: '2025-09-22', end: '2025-10-30'}]},
      ],
    }
  },
  computed: {
    headers() {
      return [
        { text: 'Odmiana', value: 'odmiana' },
        ...this.months.map(m => ({ text: m.text, value: m.value })),
        
      ];
    }
  },
  methods: {
    getDisplayStart() {
      const year = new Date().getFullYear();
      return new Date(`${year}-${this.months[0].value}-01`);
    },
    getDisplayEnd() {
      const year = new Date().getFullYear();
      const last = this.months[this.months.length - 1].value;
      const days = new Date(year, parseInt(last), 0).getDate();
      return new Date(`${year}-${last}-${days}`);
    },
    getBarStyle(startDateStr, endDateStr) {
      const msPerDay = 1000 * 60 * 60 * 24;
      const displayStart = this.getDisplayStart();
      const displayEnd = this.getDisplayEnd();
      const total = (displayEnd - displayStart) / msPerDay;

      const start = new Date(startDateStr);
      const end = new Date(endDateStr);
      const offset = (start - displayStart) / msPerDay;
      const span = (end - start) / msPerDay;

      const leftPct = (offset / total) * 100;
      const widthPct = (span / total) * 100;
      return {
        left: `${leftPct.toFixed(2)}%`,  
        width: `${widthPct.toFixed(2)}%`
      };
    }
  }
};
</script>

<style scoped>
.table-container { overflow-x: auto; }
.month-separator {
  position: absolute;           /* remove from flow, place relative to .timeline-cell */ 
  top: 0;
  bottom: 0;
  width: 1px;                   /* exactly one pixel wide */
  background-color: rgba(0,0,0,0.12);
  pointer-events: none;         /* allow clicking through */
  z-index: 1;                   /* sit above the gradient background */
}
:deep(.v-data-table) { min-width: 800px; }
.timeline-cell {
  position: relative;
  height: 32px;
  padding: 0;

  /* repeating-linear-gradient with pure percentages—no calc() expressions */  
  background: repeating-linear-gradient(
    to right,
    transparent 0%,                               /* start transparent */ 
    transparent 8.2333%,                          /* transparent up to 1/12 width */ 
    rgba(0, 0, 0, 0.12) 8.2333%,                  /* draw 1px‑equivalent line at boundary */
    rgba(0, 0, 0, 0.12) 8.3167%                   /* end the line slightly past boundary */
  );
}
.table-container :deep(th),
.table-container :deep(td) {
  border-right: 2px solid rgb(0, 0, 0);
}
/* Ensure last column has right border too */
.table-container :deep(tr) > *:last-child {
  border-right: 1px solid rgba(0, 0, 0, 0.12);
}
.table-container :deep(.v-data-table) th:first-child {
  border-right: none;
}
/* Make all bars 40% of the cell height, and increase border‑radius */
.bar {
  position: absolute;
  height: 45%;           /* 40% of the 32px row height */
  border-radius: 12px;   /* more rounded corners */
}

/* Green (blue‑class) bars go at the top */
.blue-bar {
  background-color: rgb(65, 226, 65);
  top: 10%;              /* start at half of cell height */

}

/* Orange bars go at the bottom */
.orange-bar {
  background-color: orange;
  bottom: 10%;             /* sit flush at the bottom of the cell */
}
</style>
