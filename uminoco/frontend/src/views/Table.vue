<template>
  <div>
    <v-breadcrumbs :items="breadcrumbs">
      <template v-slot:divider>
        <v-icon>mdi-chevron-right</v-icon>
      </template>
    </v-breadcrumbs>

    <!-- Dialogs -->
    <v-dialog v-model="alert" persistent max-width="320">
      <v-card>
        <v-card-title class="headline">{{this.alertTitle}}</v-card-title>
        <v-card-text v-if="this.alertMessage">{{this.alertMessage}}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="green darken-1" text @click="alert = false">Close</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Error -->
    <div v-if="failer">
      <v-container>
        <v-alert
        type="error"
        prominent
        border="left">
          <h3 class="headline">{{failer.error}}</h3>
          {{failer.message}}
        </v-alert>
      </v-container>
    </div>

    <!-- content  -->
    <div v-else-if="tableData">
      <v-container>
        <v-row no-gutters align="stretch">
          <v-col key="1">
            <v-card class="mx-auto" tile height="100%">
              <v-card-text>
                <div>Table</div>
                <p class="display-1 text--primary">{{tableData.name}}</p>
                <div v-if="this.tableCreated" >Created at: {{this.tableCreated}} </div>
                <div v-if="this.grebeSourceId" >Grebe Source ID: {{this.grebeSourceId}} </div>
              </v-card-text>
            </v-card>
          </v-col>

          <v-col key="2" sm="12" md="5" align-content="stretch">
            <v-card class="pa-0" tile height="100%" width="100%">
              <v-card-text>
                <cal-heatmap :tableName="tableName"></cal-heatmap>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>
      </v-container>

      <v-container>
        <v-row no-gutters align="stretch">
          <v-col key="1" cols="12" sm="3">
            <v-card class="pa-0" tile height="100%"><v-card-text><div>Rows</div><p class="headline mb-1">{{tableData.total_rows}}</p></v-card-text></v-card>
          </v-col>
          <v-col key="2" cols="12" sm="3">
            <div v-if="this.lastUpdated">
              <v-card class="pa-0" tile height="100%"><v-card-text><div>Last Updated</div><p class="headline mb-1">{{ lastUpdated }}</p></v-card-text></v-card>
            </div>
          </v-col>
          <v-col key="1" cols="12" sm="3">
            <v-card class="pa-0" tile height="100%"><v-card-text><div>Bytes[MB]</div><p class="headline mb-1">{{(tableData.total_bytes/1024.0/1024.0).toFixed(4)}}</p></v-card-text></v-card>
          </v-col>
          <v-col key="2" cols="12" sm="3">
            <v-card class="pa-0" tile height="100%"><v-card-text><div>Compression Ratio</div><div class="headline mb-1">{{(tableData.compression_ratio * 100).toFixed(2)}} %</div></v-card-text></v-card>
          </v-col>
        </v-row>

      </v-container>

      <!-- Data table -->
      <v-container>
        <v-card>
          <v-card-title>
            Columns
            <v-spacer></v-spacer>
            <v-text-field
              v-model="search"
              append-icon="mdi-magnify"
              label="Search"
              single-line
              hide-details
            ></v-text-field>
          </v-card-title>

          <v-data-table class='pa-2'  dense disable-pagination hide-default-footer :headers="columnsHeader" :items="tableData.columns" :search="search" no-data-text="Data not found...">
            <template v-slot:[`item.compression_ratio`]="{item}" >
              {{(item.compression_ratio * 100).toFixed(2)}} %
            </template>

            <template v-slot:[`item.recent_value`]="{item}" >
              {{ formatRecentValue(item) }}
            </template>

          </v-data-table>
        </v-card>
      </v-container>

      <!-- Rename/Delete Buttons -->
      <v-container>
        <v-row justify="end">
          <v-col class="text-center" cols="6" offset-sm="6" sm="3">
            <v-btn color="warning" @click="renameDialog = true; renameNewTableName = tableName">Rename Table</v-btn>
          </v-col>
          <v-col class="text-center" cols="6" sm="3">
            <v-btn color="error" @click="deleteDialog = true; deleteTableName = ''">Delete Table</v-btn>
          </v-col>
        </v-row>
      </v-container>

      <!-- Rename Dialog -->
      <v-dialog v-model="renameDialog" max-width="500px">
        <v-card>
          <v-toolbar flat color="warning">
            <v-icon>mdi-table-large</v-icon>
            <v-toolbar-title class="pa-2">Rename Table</v-toolbar-title>
          </v-toolbar>

          <v-card-title>Are you sure that rename the table name?</v-card-title>
          <v-card-text>
            <v-text-field v-model="renameNewTableName" color="black" label="New Table Name" :placeholder="tableName"></v-text-field>
          </v-card-text>
          <v-card-actions>
            <v-btn color="warning" :disabled="!renameNewTableName" @click="postRename(tableName, renameNewTableName)">
              RENAME
            </v-btn>
            <v-btn color="primary" text @click="renameDialog = false">
              Cancel
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

      <!-- Delete Dialog -->
      <v-dialog v-model="deleteDialog" max-width="500px">
        <v-card>
          <v-toolbar flat color="error">
            <v-icon>mdi-table-large</v-icon>
            <v-toolbar-title class="pa-2">Delete Table</v-toolbar-title>
          </v-toolbar>

          <v-card-title>Are you sure that Delete(Drop) the table name ?</v-card-title>
          <v-card-text>
            Please re-type this table name '{{this.tableName}}'
            <v-text-field v-model="deleteTableName" color="black" label="Table Name" :placeholder="tableName"></v-text-field>
          </v-card-text>
          <v-card-actions>
            <v-btn color="error" :disabled="deleteTableName != tableName" @click="postDeleteTable(deleteTableName)">
              Delete(DROP)
            </v-btn>
            <v-btn color="primary" text @click="deleteDialog = false">
              Cancel
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

    </div>

    <div v-else> <!-- tableData is null -->
      <v-data-table
        hide-default-header
        hide-default-footer
        loading
        loading-text="Loading... Please wait"
      />
    </div>

  </div>
</template>

<script>
import Clickhouse from '@/api/clickhouse.js'
import CalHeatmap from '@/components/CalHeatmap.vue'

export default {
  components: {
    CalHeatmap
  },

  data: () => ({
    breadcrumbs: [
      { text: 'Home', exact: true, disabled: false, to: { name: 'Home' } },
      { text: 'Tables', exact: true, disabled: false, to: { name: 'Tables' } },
      { text: 'loading...', exact: true, disabled: true }
    ],
    tableData: null,
    search: '',
    failer: null,
    columnsHeader: [
      { text: 'Name', sortable: true, value: 'name' },
      { text: 'Type', sortable: true, value: 'type' },
      { text: 'Size [KB]', sortable: true, value: 'data_compressed_bytes' },
      { text: 'Compression Ratio', sortable: true, value: 'compression_ratio' },
      { text: 'Recent Value', sortable: false, value: 'recent_value' }
    ],

    alert: false,
    alertTitle: '',
    alertMessage: '',

    deleteDialog: false,
    deleteTableName: '',

    renameDialog: false,
    renameNewTableName: ''
  }),

  computed: {
    tableCreated() {
      const createAt = this.tableData?.grebe_schema?.__create_at
      if (!createAt) return null
      var moment = require('moment')
      return moment(new Date(createAt)).format('YYYY-MM-DD HH:mm:ss')
    },
    lastUpdated() {
      const d = this.tableData.columns.find(e => e.name === '__create_at').recent_value
      return new Date(d).toLocaleString()
    },
    grebeSourceId() {
      return this.tableData?.grebe_schema?.source_id
    }
  },

  props: ['tableName'],

  watch: {
    tableName: {
      handler() {
        this.breadcrumbs[2] = { text: this.tableName, disabled: true }

        Clickhouse.tableDetail(this.tableName, res => {
          this.tableData = res
        },
        err => {
          this.failer = { error: err, message: err?.response?.data?.message }
        })
      },
      immediate: true
    }
  },

  methods: {
    postRename(currentTableName, newTableName) {
      this.renameDialog = false

      Clickhouse.renameTable(currentTableName, newTableName, res => {
        this.tableData = null
        this.$router.push({ name: 'Table', params: { tableName: newTableName } })
      },
      err => {
        this.alertTitle = 'Faild to rename table...'
        this.alertMessage = err?.response?.data?.message
        this.alert = true
      })
    },
    postDeleteTable(tableName) {
      this.renameDialog = false

      Clickhouse.dropTable(tableName, res => {
        this.$router.push({ name: 'Tables' })
      },
      err => {
        this.alertTitle = 'Faild to delete table...'
        this.alertMessage = err?.response?.data?.message
        this.alert = true
      })
    },
    formatRecentValue(item) {
      if (item.recent_value == null) return 'NULL'

      if (item.type.includes('DateTime')) {
        var moment = require('moment')
        if (Array.isArray(item.recent_value)) {
          return item.recent_value.map(d => moment(new Date(d)).format('YYYY-MM-DD HH:mm:ss'))
        } else {
          return moment(new Date(item.recent_value)).format('YYYY-MM-DD HH:mm:ss')
        }
      }
      return item.recent_value
    }
  }
}
</script>
