<template>
  <div class="table-properties">
    <div v-if="error" class="alert-wrapper">
      <div class="alert alert-danger">
        {{error.message}}
      </div>
    </div>
    <template  v-else>
      <div class="table-properties-header">
        <div class="nav-pills" v-if="pills.length > 1">
          <a 
            v-for="(pill) in pills"
            :key="pill.id"
            class="nav-pill"
            :class="{active: pill.id === activePill}"
            :title="pill.dirty ? 'Unsaved Changes' : ''"
            @click.prevent="activePill = pill.id"
          >
            {{pill.name}} {{pill.dirty ? '*' : ''}}
          </a>
        </div>
      </div>
      <div v-if="loading" class="table-properties-loading">
        <x-progressbar></x-progressbar>
      </div>
      <div class="table-properties-wrap" v-if="table">
        <component
          class="schema-builder"
          :is="pill.component"
          :table="table"
          :primaryKeys="primaryKeys"
          :tabState="pill"
          :properties="properties"
          :connection="connection"
          :active="pill.id === activePill && active"
          v-show="pill.id === activePill"
          v-for="(pill) in pills" 
          :key="pill.id"
          @actionCompleted="refresh"
          @refresh="refresh"
        >
          <template v-slot:footer>
            <div class="statusbar-info col flex expand">
              <x-button @click.prevent="openData" class="btn btn-flat btn-icon end" title="View Data">
                Data <i class="material-icons">north_east</i>
              </x-button>
              <template v-if="properties">
                <span class="statusbar-item" :title="totalRecords === null ? 'Loading Total Records...' : `Approximately ${totalRecords} Records`">
                  <i class="material-icons">list_alt</i>
                  <span v-if="totalRecords === null">Loading...</span>
                  <span v-else>~{{totalRecords.toLocaleString()}}</span>
                </span>
                <span class="statusbar-item" v-if="humanSize !== null" :title="`Table Size ${humanSize}`">
                  <i class="material-icons">aspect_ratio</i>
                  <span>{{humanSize}}</span>
                </span>
                <span class="statusbar-item" v-if="humanIndexSize !== null" :title="`Index Size ${humanIndexSize}`">
                  <i class="material-icons">location_searching</i>
                  <span>{{humanIndexSize}}</span>
                </span>
                <span class="statusbar-item" v-if="!editable" title="Only tables can be edited.">
                  <i class="material-icons-outlined">report_problem</i> Read Only
                </span>
              </template>
            </div>
          </template>

          <template v-slot:actions >
            <x-button class="actions-btn btn btn-flat" title="Actions">
              <i class="material-icons">settings</i>
              <i class="material-icons">arrow_drop_down</i>
              <x-menu>
                <x-menuitem @click.prevent="refresh">
                  <x-label>Refresh</x-label>
                </x-menuitem>
                <x-menuitem @click.prevent="openTable">
                  <x-label>View Data</x-label>
                </x-menuitem>
                <hr v-if="dev">
                <x-menuitem v-if="dev" @click.prevent="triggerError">
                  <x-label>[DEV] Toggle Error</x-label>
                </x-menuitem>
                <x-menuitem v-if="dev" @click.prevent="loading = !loading">
                  <x-label>[DEV] Toggle Loading</x-label>
                </x-menuitem>
              </x-menu>
            </x-button>
          </template>
        </component>
      </div>
    </template>
  </div>
</template>

<script>
import Tabulator from 'tabulator-tables'
import Statusbar from './common/StatusBar'
import TableSchemaVue from './tableinfo/TableSchema.vue'
import TableIndexesVue from './tableinfo/TableIndexes.vue'
import TableRelationsVue from './tableinfo/TableRelations.vue'
import TableTriggersVue from './tableinfo/TableTriggers.vue'
import { format as humanBytes } from 'bytes'
import platformInfo from '../common/platform_info'
import { AppEvent } from '@/common/AppEvent'
import { mapState } from 'vuex'
import rawLog from 'electron-log'

const log = rawLog.scope('TabTableProperties')
export default {
  props: ["connection", "tabId", "active", "tab", "table"],
  components: { Statusbar },
  data() {
    return {
      initialized: false,
      dev: platformInfo.isDevelopment,
      loading: true,
      error: null,
      primaryKeys: [],
      properties: {},
      totalRecords: null,
      dirtyPills: {},
      rawPills: [
        {
          id: 'schema',
          name: "Columns",
          needsProperties: false,
          component: TableSchemaVue,
          dirty: false,
        },
        {
          id: 'indexes',
          name: "Indexes",
          tableOnly: true,
          needsProperties: true,
          component: TableIndexesVue,
          dirty: false,
        },
        {
          id: 'relations',
          name: "Relations",
          tableOnly: true,
          needsProperties: true,
          component: TableRelationsVue,
          dirty: false,
        },
        {
          id: 'triggers',
          name: "Triggers",
          tableOnly: true,
          needsProperties: true,
          component: TableTriggersVue,
          dirty: false
        }
      ],
      activePill: 'schema', // the only tab that is always there.
      tableSchema: null,
      tableIndexes: null,
      tableRelations: null,
      tableTriggers: null,
      actualTableHeight: "100%",
    }
  },
  watch: {
    unsavedChanges() {
      this.tab.unsavedChanges = this.unsavedChanges
    },
    shouldInitialize() {
      if (this.shouldInitialize) this.initialize()
    }
  },
  computed: {
    ...mapState(['tables', 'tablesInitialLoaded']),
    shouldInitialize() {
      // TODO (matthew): Move this to the wrapper TabWithTable
      return this.tablesInitialLoaded && this.active && !this.initialized
    },
    editable() {
      return this.table.entityType === 'table' && !!this.primaryKeys.length
    },
    unsavedChanges() {
      return this.pills.filter((p) => p.dirty).length > 0
    },
    pills() {
      if (!this.table) return []
      const isTable = this.table.entityType === 'table'
      return this.rawPills.filter((p) => {

        if (!this.properties) {
          if (p.needsProperties) {
            return false
          }
        }
        
        if (p.needsProperties && !this.connection.supportedFeatures().properties) {
          return false
        }
        if(p.tableOnly) {
          return isTable
        }
        return true
      })
    },
    humanSize() {
      return humanBytes(this.properties.size)
    },
    humanIndexSize() {
      return humanBytes(this.properties.indexSize)
    },
  },
  methods: {
    close() {
      this.$root.$emit(AppEvent.closeTab)
    },
    openData() {
      this.$root.$emit(AppEvent.loadTable, { table: this.table })
    },
    triggerError() {
      // this is for dev only
      this.error = new Error("Something went wrong")
    },
    async fetchTotalRecords() {
      try {
        this.totalRecords = await this.connection.getTableLength(this.table.name, this.table.schema)
      } catch (ex) {
        console.error("unable to fetch total records", ex)
        this.totalRecords = 0
      } 
    },
    async initialize() {
      log.info("initializing")
      this.initialized = true
      await this.refresh()
    },
    async refresh() {
      if (!this.table) return;
      log.info("refresh")
      this.loading = true
      this.error = null
      // this.properties = null
      this.fetchTotalRecords()
      try {
        this.primaryKeys = await this.connection.getPrimaryKeys(this.table.name, this.table.schema)
        if (this.table.entityType === 'table') {
          this.properties = await this.connection.getTableProperties(this.table.name, this.table.schema)
        }
        this.loading = false
      } catch (ex) {
        this.error = ex
      } finally {
        log.info("setting loaded = false")
        this.loading = false
      }
    },
    async openTable() {
      this.$root.$emit("loadTable", { table: this.table })
    }
  },
  async mounted() {
    if (this.shouldInitialize) this.initialize()
  }
}
</script>