<template>
  <div>
    <v-tabs
      v-model="logsTab"
      class="mb-2 error-log-tabs"
      :slider-color="getTabItemsColor(currentTab.key)"
      :style="{ borderBottomColor: getTabItemsBorderColor(currentTab.key) }"
      @change="onTabChange"
    >
      <v-tab
        v-for="(tabItem, index) in tabs"
        :key="tabItem.key"
        :class="[
          `error-log-tab--${tabItem.key}`,
          { 'error-log-tab--active': logsTab === index }
        ]"
      >
        <span>{{ tabItem.label }}</span>
        <v-chip
          x-small
          :color="getTabItemsColor(tabItem.key)"
          dark
          class="ml-2"
        >
          {{ getTabItemsCount(tabItem.key) }}
        </v-chip>
      </v-tab>
    </v-tabs>
    <v-card outlined class="mb-3 pa-3 filter-panel">
      <v-row dense align="center">
        <v-col cols="12" md="4">
          <v-text-field
            v-model="searchQuery"
            :prepend-inner-icon="icons.mdiMagnify"
            label="Search logs"
            clearable
            dense
            outlined
            hide-details
            @input="resetMobilePage"
          />
        </v-col>
        <v-col cols="12" sm="6" md="2">
          <v-select
            v-model="selectedApplication"
            :items="applicationOptions"
            label="Application"
            clearable
            dense
            outlined
            hide-details
            @input="resetMobilePage"
          />
        </v-col>
        <v-col v-if="currentTab.key === 'services'" cols="12" sm="6" md="2">
          <v-select
            v-model="selectedService"
            :items="serviceOptions"
            label="Service"
            clearable
            dense
            outlined
            hide-details
            @input="resetMobilePage"
          />
        </v-col>
        <v-col cols="12" sm="6" md="2">
          <v-text-field
            v-model="dateTimeFrom"
            label="From"
            type="datetime-local"
            clearable
            dense
            outlined
            hide-details
            @input="resetMobilePage"
          />
        </v-col>
        <v-col cols="12" sm="6" md="2">
          <v-text-field
            v-model="dateTimeTo"
            label="To"
            type="datetime-local"
            clearable
            dense
            outlined
            hide-details
            @input="resetMobilePage"
          />
        </v-col>
        <v-col v-if="hasActiveFilters" cols="12" class="d-flex justify-end">
          <v-btn small text color="primary" @click="clearFilters">
            <v-icon left small>
              {{ icons.mdiFilterRemoveOutline }}
            </v-icon>
            Clear filters
          </v-btn>
        </v-col>
      </v-row>
      <v-row v-if="$vuetify.breakpoint.smAndDown" dense align="center" class="mt-2">
        <v-col cols="9">
          <v-select
            v-model="mobileSortBy"
            :items="mobileSortOptions"
            label="Sort by"
            dense
            outlined
            hide-details
            @change="resetMobilePage"
          />
        </v-col>
        <v-col cols="3" class="d-flex justify-end">
          <v-btn outlined color="primary" aria-label="Toggle sort direction" @click="toggleMobileSortDirection">
            <v-icon>
              {{ mobileSortDescending ? icons.mdiSortDescending : icons.mdiSortAscending }}
            </v-icon>
          </v-btn>
        </v-col>
      </v-row>
    </v-card>

    <div v-if="$vuetify.breakpoint.smAndDown">
      <div class="d-flex flex-wrap justify-end mb-3 mobile-list-actions">
        <v-btn small color="primary" dark @click="fetchCurrentList(false)">
          <v-icon left small>
            {{ icons.mdiRefresh }}
          </v-icon>
          Refresh
        </v-btn>
        <v-btn small color="red" dark @click="clearAll">
          <v-icon left small>
            {{ icons.mdiDeleteAlertOutline }}
          </v-icon>
          Remove all
        </v-btn>
        <v-btn v-if="canGenerateCurrentTab" small color="indigo" dark @click="generateError">
          <v-icon left small>
            {{ icons.mdiPlaylistPlus }}
          </v-icon>
          Generate
        </v-btn>
      </div>

      <v-progress-linear v-if="isLoading" indeterminate color="primary" class="mb-3" />
      <v-alert v-else-if="!mobileDisplayedItems.length" type="info" outlined>
        No logs match the selected filters.
      </v-alert>
      <template v-else>
        <v-card
          v-for="item in mobilePaginatedItems"
          :key="item.ID"
          outlined
          class="mb-4 mobile-log-card"
        >
          <v-card-title class="align-start mobile-log-card__title">
            <span class="text-break">{{ item.Message }}</span>
            <v-chip small color="primary" dark>
              {{ item.Counter || 0 }} occurrences
            </v-chip>
          </v-card-title>
          <v-card-subtitle class="pb-2">
            <div class="d-flex flex-wrap align-center mobile-log-card__meta">
              <v-chip small outlined color="primary">
                {{ item.AppName || 'Unknown application' }}
              </v-chip>
              <v-chip v-if="item.Service" small outlined color="teal">
                {{ item.Service }}
              </v-chip>
              <span>{{ formatDateTime(item.Time) }}</span>
            </div>
          </v-card-subtitle>
          <v-divider />
          <v-card-text>
            <div class="mobile-log-card__details mb-4">
              <div><strong>ID:</strong> {{ item.ID }}</div>
              <div><strong>File:</strong> <span class="text-break">{{ item.File }}</span></div>
              <div><strong>Line:</strong> {{ item.Line }}</div>
            </div>

            <div class="mobile-log-section mb-4">
              <div class="d-flex align-center justify-space-between mobile-log-section__header">
                <v-btn
                  small
                  text
                  class="text-none mobile-log-section__toggle"
                  :aria-expanded="isMobileSectionExpanded(item, 'log')"
                  @click="toggleMobileSection(item, 'log')"
                >
                  <v-icon left small>
                    {{ isMobileSectionExpanded(item, 'log') ? icons.mdiChevronUp : icons.mdiChevronDown }}
                  </v-icon>
                  <strong>Log</strong>
                </v-btn>
                <v-btn small text color="primary" @click.stop="copyLog(item)">
                  <v-icon left small>
                    {{ icons.mdiContentCopy }}
                  </v-icon>
                  Copy log
                </v-btn>
              </div>
              <v-expand-transition>
                <pre v-show="isMobileSectionExpanded(item, 'log')" class="mobile-log-content">{{ item.Stack || 'No log captured.' }}</pre>
              </v-expand-transition>
            </div>

            <div class="mobile-log-section">
              <div class="d-flex align-center justify-space-between mobile-log-section__header">
                <v-btn
                  small
                  text
                  class="text-none mobile-log-section__toggle"
                  :aria-expanded="isMobileSectionExpanded(item, 'request')"
                  @click="toggleMobileSection(item, 'request')"
                >
                  <v-icon left small>
                    {{ isMobileSectionExpanded(item, 'request') ? icons.mdiChevronUp : icons.mdiChevronDown }}
                  </v-icon>
                  <strong>Request</strong>
                </v-btn>
                <v-btn small text color="primary" :disabled="!item.Request" @click.stop="copyRequest(item)">
                  <v-icon left small>
                    {{ icons.mdiContentCopy }}
                  </v-icon>
                  Copy request
                </v-btn>
              </div>
              <v-expand-transition>
                <pre v-show="isMobileSectionExpanded(item, 'request')" class="mobile-log-content">{{ item.Request || 'No request captured.' }}</pre>
              </v-expand-transition>
            </div>
          </v-card-text>
          <v-divider />
          <v-card-actions class="mobile-log-card__actions">
            <v-btn
              small
              text
              class="text-none"
              :color="getTicketLink(item) ? 'primary' : 'orange'"
              :loading="jiraActionLoadingByItem[item.ID]"
              @click="onJiraAction(item)"
            >
              {{ getTicketLink(item) ? 'Open Ticket' : 'Create Ticket' }}
            </v-btn>
            <v-spacer />
            <v-btn small text color="red" @click="deleteItem(item)">
              <v-icon left small>
                {{ icons.mdiDelete }}
              </v-icon>
              Delete
            </v-btn>
          </v-card-actions>
        </v-card>
      </template>

      <v-pagination
        v-if="mobilePageCount > 1"
        v-model="mobilePage"
        :length="mobilePageCount"
        :total-visible="5"
        class="mb-4"
      />
    </div>

    <v-data-table
      v-else
      :headers="tableHeaders"
      :items="displayedItems"
      :loading="isLoading"
      loading-text="Loading... Please wait"
      class="elevation-1"
      :items-per-page="50"
      fixed-header
      :height="$vuetify.breakpoint.height - 200"
      sort-by="Counter"
      sort-desc
      must-sort
      single-expand
      show-expand
      item-key="ID"
      :expanded.sync="expanded"
      :hide-default-footer="!displayedItems.length"
      :footer-props="{ 'items-per-page-options': [50, 100, 1000, -1] }"
      @click:row="onClickRow"
    >
      <template #top>
        <v-toolbar flat>
          <v-spacer />
          <div class="text-center d-flex align-center justify-space-around">
            <v-tooltip bottom color="grey darken-1" content-class="py-1">
              <template v-slot:activator="{ on, attrs }">
                <v-btn
                  color="primary"
                  small
                  dark
                  class="mr-2"
                  v-bind="attrs"
                  v-on="on"
                >
                  <v-icon size="18px" @click="fetchCurrentList(false)">
                    {{ icons.mdiRefresh }}
                  </v-icon>
                </v-btn>
              </template>
              <span class="white--text text-caption">Refresh</span>
            </v-tooltip>
            <v-tooltip bottom color="grey darken-1" content-class="py-1">
              <template v-slot:activator="{ on, attrs }">
                <v-btn
                  color="red"
                  small
                  dark
                  class="mr-2"
                  v-bind="attrs"
                  v-on="on"
                >
                  <v-icon size="18px" @click="clearAll">
                    {{ icons.mdiDeleteAlertOutline }}
                  </v-icon>
                </v-btn>
              </template>
              <span class="white--text text-caption">Remove all {{ currentTab.label.toLowerCase() }}</span>
            </v-tooltip>
            <v-tooltip bottom color="grey darken-1" content-class="py-1">
              <template v-slot:activator="{ on, attrs }">
                <v-btn
                  v-if="canGenerateCurrentTab"
                  color="indigo"
                  small
                  dark
                  class="mr-2"
                  v-bind="attrs"
                  v-on="on"
                >
                  <v-icon size="18px" @click="generateError">
                    {{ icons.mdiPlaylistPlus }}
                  </v-icon>
                </v-btn>
              </template>
              <span class="white--text text-caption">Generate sample entry</span>
            </v-tooltip>
          </div>
        </v-toolbar>
      </template>
      <template #item.actions="{ item }">
        <div class="actions-cell">
          <v-btn
            x-small
            text
            class="text-none ticket-action-btn"
            :color="getTicketLink(item) ? 'primary' : 'orange'"
            :loading="jiraActionLoadingByItem[item.ID]"
            @click.stop="onJiraAction(item)"
          >
            {{ getTicketLink(item) ? 'Ticket' : 'Create Ticket' }}
          </v-btn>
          <v-tooltip bottom color="grey darken-1" content-class="py-1">
            <template v-slot:activator="{ on, attrs }">
              <v-icon
                small
                v-bind="attrs"
                @click.stop="deleteItem(item)"
                v-on="on"
              >
                {{ icons.mdiDelete }}
              </v-icon>
            </template>
            <span class="white--text text-caption">Delete {{ getCurrentTabItemLabel() }}</span>
          </v-tooltip>
        </div>
      </template>
      <template #item.lp="{ index }">
        {{ ++index }}
      </template>
      <template #item.Time="{ item }">
        <span :id="`err-${item.ID}`">
          {{ formatDateTime(item.Time) }}
        </span>
      </template>
      <template #item.Description="{ item }">
        <div class="px-1 py-2 text-break">
          <div class="text-subtitle-2">
            {{ item.File }}
            <span class="text-caption">(Line: {{ item.Line }})</span>
          </div>
          <div class="error--text">
            {{ item.Message }}
          </div>
        </div>
      </template>
      <template #expanded-item="{ headers, item }">
        <td :colspan="headers.length">
          <v-tabs v-model="tab" background-color="transparent">
            <v-tab>
              <v-subheader class="text-capitalize">
                Stack
              </v-subheader>
            </v-tab>
            <v-tab :disabled="!item.Request">
              <v-subheader class="text-capitalize">
                Request
              </v-subheader>
            </v-tab>
          </v-tabs>
          <v-tabs-items v-model="tab" class="mb-4 stack-window">
            <v-tab-item>
              <div class="pb-6 stack">
                {{ item.Stack }}
              </div>
            </v-tab-item>
            <v-tab-item>
              <div class="pb-6 stack">
                {{ item.Request }}
              </div>
            </v-tab-item>
          </v-tabs-items>
        </td>
      </template>
    </v-data-table>
  </div>
</template>

<script lang="ts">
import {
  mdiChevronDown,
  mdiChevronUp,
  mdiContentCopy,
  mdiDelete,
  mdiDeleteAlertOutline,
  mdiEyeOutline,
  mdiFilterRemoveOutline,
  mdiMagnify,
  mdiPlaylistPlus,
  mdiRefresh,
  mdiSortAscending,
  mdiSortDescending
} from '@mdi/js'
import { Component, mixins } from 'nuxt-property-decorator'
import { ApiUtilities } from '~/components/mixins/Global'

@Component
export default class ErrorsLog extends mixins(ApiUtilities) {
  fetch () {
    this.setHeaders()
    this.fetchCounters()
    this.fetchCurrentList()
  }

  icons = {
    mdiChevronDown,
    mdiChevronUp,
    mdiEyeOutline,
    mdiContentCopy,
    mdiDelete,
    mdiRefresh,
    mdiDeleteAlertOutline,
    mdiFilterRemoveOutline,
    mdiMagnify,
    mdiPlaylistPlus,
    mdiSortAscending,
    mdiSortDescending
  }

  headers:any = []
  tabs:any = [
    {
      key: 'errors',
      label: 'Errors',
      endpoints: {
        list: '/error-log/errors/',
        removePrefix: '/error-log/errors/remove/',
        removeAll: '/error-log/errors/remove-all/',
        createTicketPrefix: '/error-log/errors/create-jira-ticket/',
        generate: '/error-log/panic/'
      }
    },
    {
      key: 'services',
      label: 'Service logs',
      endpoints: {
        list: '/error-log/services/',
        removePrefix: '/error-log/services/remove/',
        removeAll: '/error-log/services/remove-all/',
        createTicketPrefix: '/error-log/services/create-jira-ticket/',
        generate: '/dev/create-service-test-logs/'
      }
    },
    {
      key: 'warnings',
      label: 'Warnings',
      endpoints: {
        list: '/error-log/warnings/',
        removePrefix: '/error-log/warnings/remove/',
        removeAll: '/error-log/warnings/remove-all/',
        createTicketPrefix: '/error-log/warnings/create-jira-ticket/',
        generate: '/error-log/panic/'
      }
    },
    {
      key: 'missingTranslations',
      label: 'Missing translations',
      endpoints: {
        list: '/error-log/missing-translations/',
        removePrefix: '/error-log/missing-translations/remove/',
        removeAll: '/error-log/missing-translations/remove-all/',
        createTicketPrefix: '/error-log/missing-translations/create-jira-ticket/',
        generate: '/error-log/panic/'
      }
    }
  ]

  itemsByTab:any = {
    errors: [],
    services: [],
    warnings: [],
    missingTranslations: []
  }

  countersByTab:any = {
    errors: 0,
    services: 0,
    warnings: 0,
    missingTranslations: 0
  }

  expanded:any = []
  jiraActionLoadingByItem:any = {}
  logsTab:number = 0
  tab:number = 0
  isLoading:boolean = false
  searchQuery:string|null = ''
  selectedApplication:string|null = null
  selectedService:string|null = null
  dateTimeFrom:string|null = null
  dateTimeTo:string|null = null
  mobileSortBy:string = 'Time'
  mobileSortDescending:boolean = true
  mobilePage:number = 1
  mobileItemsPerPage:number = 10
  mobileExpandedSections:any = {}

  get currentTab () {
    return this.tabs[this.logsTab] || this.tabs[0]
  }

  get displayedItems () {
    const items = this.itemsByTab[this.currentTab.key] || []
    const searchQuery = (this.searchQuery || '').trim().toLowerCase()
    const fromTimestamp = this.dateTimeFrom ? new Date(this.dateTimeFrom).getTime() : null
    const toTimestamp = this.dateTimeTo ? new Date(this.dateTimeTo).getTime() : null

    return items.filter((item:any) => {
      if (this.selectedApplication && item.AppName !== this.selectedApplication) {
        return false
      }

      if (this.currentTab.key === 'services' && this.selectedService && item.Service !== this.selectedService) {
        return false
      }

      if (searchQuery) {
        const searchableText = [
          item.ID,
          item.AppName,
          item.Service,
          item.File,
          item.Line,
          item.Message,
          item.Stack,
          item.Request,
          item.Time
        ].map(value => String(value || '').toLowerCase()).join(' ')

        if (!searchableText.includes(searchQuery)) {
          return false
        }
      }

      const itemTimestamp = this.getLogTimestamp(item.Time)
      if ((fromTimestamp !== null || toTimestamp !== null) && Number.isNaN(itemTimestamp)) {
        return false
      }

      if (fromTimestamp !== null && !Number.isNaN(fromTimestamp) && itemTimestamp < fromTimestamp) {
        return false
      }

      if (toTimestamp !== null && !Number.isNaN(toTimestamp) && itemTimestamp > toTimestamp) {
        return false
      }

      return true
    })
  }

  get mobileDisplayedItems () {
    const items = [...this.displayedItems]
    const direction = this.mobileSortDescending ? -1 : 1

    return items.sort((firstItem:any, secondItem:any) => {
      let firstValue:any = firstItem[this.mobileSortBy]
      let secondValue:any = secondItem[this.mobileSortBy]

      if (this.mobileSortBy === 'Time') {
        firstValue = this.getLogTimestamp(firstValue)
        secondValue = this.getLogTimestamp(secondValue)
      } else if (this.mobileSortBy === 'Counter') {
        firstValue = Number(firstValue || 0)
        secondValue = Number(secondValue || 0)
      } else {
        firstValue = String(firstValue || '').toLowerCase()
        secondValue = String(secondValue || '').toLowerCase()
      }

      if (firstValue === secondValue) {
        return 0
      }

      return firstValue > secondValue ? direction : -direction
    })
  }

  get mobilePageCount () {
    return Math.ceil(this.mobileDisplayedItems.length / this.mobileItemsPerPage)
  }

  get mobilePaginatedItems () {
    const page = Math.min(this.mobilePage, Math.max(this.mobilePageCount, 1))
    const start = (page - 1) * this.mobileItemsPerPage

    return this.mobileDisplayedItems.slice(start, start + this.mobileItemsPerPage)
  }

  get mobileSortOptions () {
    const options = [
      { text: 'Time', value: 'Time' },
      { text: 'Occurrences', value: 'Counter' },
      { text: 'Application', value: 'AppName' },
      { text: 'Message', value: 'Message' }
    ]

    if (this.currentTab.key === 'services') {
      options.push({ text: 'Service', value: 'Service' })
    }

    return options
  }

  get applicationOptions () {
    const applications = (this.itemsByTab[this.currentTab.key] || [])
      .map((item:any) => item.AppName)
      .filter((application:string) => !!application)

    return applications
      .filter((application:string, index:number) => applications.indexOf(application) === index)
      .sort()
  }

  get hasActiveFilters () {
    return !!(
      this.searchQuery ||
      this.selectedApplication ||
      this.selectedService ||
      this.dateTimeFrom ||
      this.dateTimeTo
    )
  }

  get tableHeaders () {
    if (this.currentTab.key !== 'services') {
      return this.headers
    }

    return [
      ...this.headers.slice(0, 3),
      { text: 'Service', value: 'Service', width: '150px' },
      ...this.headers.slice(3)
    ]
  }

  get serviceOptions () {
    const services = (this.itemsByTab.services || [])
      .map((item:any) => item.Service)
      .filter((service:string) => !!service)

    return services
      .filter((service:string, index:number) => services.indexOf(service) === index)
      .sort()
  }

  get canGenerateCurrentTab () {
    return this.currentTab.key === 'errors' || this.currentTab.key === 'services'
  }

  getTabItemsCount (tabKey:string) {
    return this.countersByTab[tabKey] ?? (this.itemsByTab[tabKey] || []).length
  }

  getTabItemsColor (tabKey:string) {
    const colors:any = {
      errors: 'red',
      services: 'teal',
      warnings: 'blue',
      missingTranslations: 'purple'
    }
    return colors[tabKey] || 'primary'
  }

  getTabItemsBorderColor (tabKey:string) {
    const colors:any = {
      errors: '#f44336',
      services: '#009688',
      warnings: '#2196f3',
      missingTranslations: '#9c27b0'
    }
    return colors[tabKey] || '#1976d2'
  }

  getCurrentTabItemLabel () {
    const labels:any = {
      errors: 'error',
      services: 'service log',
      warnings: 'warning',
      missingTranslations: 'missing translation'
    }
    return labels[this.currentTab.key] || 'entry'
  }

  getTicketLink (item:any) {
    return item?.ticketLink || item?.TicketLink || ''
  }

  formatDateTime (time:string) {
    const date = new Date(this.getLogTimestamp(time))
    if (Number.isNaN(date.getTime())) {
      return time
    }

    const pad = (value:number) => String(value).padStart(2, '0')

    return `${pad(date.getDate())}.${pad(date.getMonth() + 1)}.${date.getFullYear()} ${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`
  }

  getLogTimestamp (time:string) {
    const normalizedTime = time || ''
    const directTimestamp = new Date(normalizedTime).getTime()
    if (!Number.isNaN(directTimestamp)) {
      return directTimestamp
    }

    const goTime = normalizedTime.match(/^(\d{4}-\d{2}-\d{2})\s+(\d{2}:\d{2}:\d{2}(?:\.\d+)?)\s+([+-]\d{2})(\d{2})/)
    if (!goTime) {
      return Number.NaN
    }

    return Date.parse(`${goTime[1]}T${goTime[2]}${goTime[3]}:${goTime[4]}`)
  }

  fetchCounters () {
    this.$axios
      .get('/error-log/counters/')
      .then((resp) => {
        const counters = resp.data || {}
        this.countersByTab = {
          errors: counters.errors || 0,
          services: counters.services || 0,
          warnings: counters.warnings || 0,
          missingTranslations: counters.missingTranslations || 0
        }
      })
      .catch(this.apiOnCatchError(''))
  }

  created () {
    if (this.$route.hash.substr(0, 5) === '#err-') {
      this.expanded = [
        { ID: this.$route.hash.substr(5) }
      ]
    }
  }

  setHeaders () {
    this.headers = [
      {
        text: '#',
        align: 'start',
        sortable: false,
        width: '50px',
        value: 'lp'
      },
      { text: 'Counter', value: 'Counter', align: 'center' },
      { text: 'Application', value: 'AppName', width: '150px' },
      { text: 'Time', value: 'Time' },
      { text: 'Description', value: 'Description', sortable: false, width: '45%' },
      { text: '', value: 'data-table-expand' },
      { text: '', value: 'actions', sortable: false, width: '240px' }
    ]
  }

  fetchCurrentList (force = false) {
    return this.getListByTabKey(this.currentTab.key, force)
  }

  getListByTabKey (tabKey:string, force = false) {
    if (!force && this.isLoading) {
      return false
    }
    const activeTab = this.tabs.find((tabItem:any) => tabItem.key === tabKey) || this.tabs[0]

    this.api()
      .get(activeTab.endpoints.list)
      .then((resp) => {
        const rows:object[] = []
        if (resp.data) {
          Object.keys(resp.data).forEach((key) => {
            rows.push({ ...resp.data[key], ID: key })
          })
        }
        this.itemsByTab[tabKey] = rows
        this.countersByTab[tabKey] = rows.length
      })
      .catch(this.apiOnCatchError)
      .then(this.apiOnFinishRequest)
  }

  onTabChange () {
    this.expanded = []
    this.tab = 0
    this.selectedApplication = null
    this.selectedService = null
    this.mobileSortBy = 'Time'
    this.mobileSortDescending = true
    this.mobileExpandedSections = {}
    this.resetMobilePage()
    this.fetchCurrentList()
  }

  resetMobilePage () {
    this.mobilePage = 1
  }

  toggleMobileSortDirection () {
    this.mobileSortDescending = !this.mobileSortDescending
    this.resetMobilePage()
  }

  getMobileSectionKey (item:any, section:string) {
    return `${this.currentTab.key}:${item.ID}:${section}`
  }

  isMobileSectionExpanded (item:any, section:string) {
    return !!this.mobileExpandedSections[this.getMobileSectionKey(item, section)]
  }

  toggleMobileSection (item:any, section:string) {
    const key = this.getMobileSectionKey(item, section)
    this.$set(this.mobileExpandedSections, key, !this.mobileExpandedSections[key])
  }

  clearFilters () {
    this.searchQuery = ''
    this.selectedApplication = null
    this.selectedService = null
    this.dateTimeFrom = null
    this.dateTimeTo = null
    this.resetMobilePage()
  }

  getLogText (item:any) {
    return [
      `ID: ${item.ID}`,
      `Application: ${item.AppName || ''}`,
      `Service: ${item.Service || ''}`,
      `Time: ${item.Time || ''}`,
      `Occurrences: ${item.Counter || 0}`,
      `File: ${item.File || ''}`,
      `Line: ${item.Line || ''}`,
      '',
      'Message:',
      item.Message || '',
      '',
      'Stack:',
      item.Stack || ''
    ].join('\n')
  }

  copyLog (item:any) {
    return this.copyText(this.getLogText(item), 'Log')
  }

  copyRequest (item:any) {
    return this.copyText(item.Request || '', 'Request')
  }

  async copyText (text:string, label:string) {
    if (!text) {
      return
    }

    try {
      if (navigator.clipboard) {
        await navigator.clipboard.writeText(text)
      } else {
        const textArea = document.createElement('textarea')
        textArea.value = text
        textArea.style.position = 'fixed'
        textArea.style.opacity = '0'
        document.body.appendChild(textArea)
        textArea.select()

        const copied = document.execCommand('copy')
        document.body.removeChild(textArea)
        if (!copied) {
          throw new Error('copy failed')
        }
      }

      this.$dialog.message.success(`${label} copied`, {
        position: 'botton-right',
        timeout: 2000
      })
    } catch (error) {
      this.$notification.show({
        type: 'error',
        message: `Could not copy ${label.toLowerCase()}`
      })
    }
  }

  async deleteItem (item:any) {
    const tabLabel = this.currentTab.label.toLowerCase().replace(/s$/, '')
    const confirm = await this.$dialog.confirm({
      title: 'Are you sure?',
      text: `Delete current ${tabLabel}?`,
      actions: {
        false: 'Cancel',
        true: 'Confirm'
      }
    })
    if (confirm) {
      this.api()
        .get(`${this.currentTab.endpoints.removePrefix}${item.ID}/`)
        .then(() => {
          this.$dialog.message.success('Deleted', {
            position: 'botton-right',
            timeout: 3000
          })
          this.fetchCurrentList(true)
          this.fetchCounters()
        })
        .catch(this.apiOnCatchError)
        .then(this.apiOnFinishRequest)
    }
  }

  onJiraAction (item:any) {
    const ticketLink = this.getTicketLink(item)
    if (ticketLink) {
      window.open(ticketLink, '_blank', 'noopener')
      return
    }

    if (this.jiraActionLoadingByItem[item.ID]) {
      return
    }

    this.$set(this.jiraActionLoadingByItem, item.ID, true)
    this.api()
      .post(`${this.currentTab.endpoints.createTicketPrefix}${item.ID}/`)
      .then((resp) => {
        const payload = resp?.data || {}
        const createdTicketLink = payload.ticketLink || payload.TicketLink || ''
        this.$set(item, 'ticketLink', createdTicketLink)
        this.$set(item, 'TicketLink', createdTicketLink)
        this.$set(item, 'ticketKey', payload.ticketKey || '')
        this.$dialog.message.success(payload.ticketKey ? `Jira ticket ${payload.ticketKey} linked` : 'Jira ticket linked', {
          position: 'botton-right',
          timeout: 3000
        })
      })
      .catch(this.apiOnCatchError)
      .then(() => {
        this.$set(this.jiraActionLoadingByItem, item.ID, false)
        this.apiOnFinishRequest()
      })
  }

  async clearAll () {
    const tabLabel = this.currentTab.label.toLowerCase()
    const confirm = await this.$dialog.confirm({
      title: 'Are you sure?',
      text: `Delete all ${tabLabel}?`,
      actions: {
        false: 'Cancel',
        true: 'Confirm'
      }
    })
    if (confirm) {
      this.api()
        .get(this.currentTab.endpoints.removeAll)
        .then(() => {
          this.fetchCurrentList(true)
          this.fetchCounters()
        })
        .catch(this.apiOnCatchError)
        .then(this.apiOnFinishRequest)
    }
  }

  generateError () {
    if (this.isLoading) {
      return false
    }
    this.api()
      .get(this.currentTab.endpoints.generate)
      .then(() => {
        this.fetchCurrentList(true)
        this.fetchCounters()
      })
      .catch(this.apiOnCatchError)
      .then(this.apiOnFinishRequest)
  }

  onClickRow (slotData?: any) {
    this.expanded = [
      { ID: slotData.ID }
    ]

    this.tab = 0

    // This doesnt work for some reason. Perhaps Vuetify docs have not been updated?
    // slotData.expand(!slotData.isExpanded)
  }
}
</script>

<style scoped lang="scss">
.error-log-tabs {
  border-bottom: 2px solid;
}

.v-tabs::v-deep {
  .v-tab.error-log-tab--active.error-log-tab--errors {
    color: #f44336 !important;
  }

  .v-tab.error-log-tab--active.error-log-tab--services {
    color: #009688 !important;
  }

  .v-tab.error-log-tab--active.error-log-tab--warnings {
    color: #2196f3 !important;
  }

  .v-tab.error-log-tab--active.error-log-tab--missingTranslations {
    color: #9c27b0 !important;
  }
}

.filter-panel {
  background: #fff;
}

.mobile-list-actions {
  gap: 8px;
}

.mobile-log-card {
  overflow: hidden;

  &__title {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    gap: 12px;
    font-size: 16px;
    line-height: 1.4;
  }

  &__meta {
    gap: 8px;
  }

  &__details {
    display: grid;
    gap: 6px;
  }

  &__actions {
    flex-wrap: wrap;
  }
}

.mobile-log-section {
  padding: 12px;
  background: #f7f7f7;
  border: 1px solid #e0e0e0;
  border-radius: 4px;

  &__header {
    min-height: 36px;
  }

  &__toggle {
    flex: 1;
    justify-content: flex-start;
    min-width: 0 !important;
  }
}

.mobile-log-content {
  max-height: 280px;
  padding: 12px;
  margin: 0;
  overflow: auto;
  color: #333;
  overflow-wrap: normal;
  word-break: normal;
  white-space: pre;
  background: #fff;
  border-radius: 4px;
}

.v-data-table::v-deep {
  table {
    table-layout: fixed;
  }
  th {
    white-space: nowrap;
  }
  .actions-cell {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    white-space: nowrap;
  }
  .ticket-action-btn {
    min-width: 120px;
    white-space: nowrap;
  }
  .text-break {
    max-width: 100%;
    overflow-wrap: anywhere;
  }
  .error--text {
    font-size: 13px;
    line-height: 1.3em;
  }
  .stack {
    white-space: pre-line;
    font-style: italic;
    min-height: 300px;
    padding: 16px;
    &-window {
      background-color: #f7ecec6e !important;
    }
  }
  .v-data-table__expanded {
    background-color: #f7ecec6e !important;
  }
  .v-data-table__wrapper {
    position: relative;
    .v-data-table__empty-wrapper {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, 0);
    }
  }
}
</style>
