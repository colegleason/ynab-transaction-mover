<template>
  <div class="container">
    <!-- vue-good-table component below -->
    <vue-good-table class="row"
      @on-selected-rows-change="selectionChanged"
      :columns="columns"
      :rows="transactions"
      :select-options="{
        enabled: true,
        selectionText: 'transactions selected',
        clearSelectionText: 'clear',
      }"
      :pagination-options="{
        enabled: true,
        mode: 'records',
        perPage: 10,
        position: 'bottom',
        perPageDropdown: [10, 50, 100, 500],
        dropdownAllowAll: true,
        nextLabel: 'next',
        prevLabel: 'prev',
        rowsPerPageLabel: 'Transactions per page',
        ofLabel: 'of',
        pageLabel: 'page', // for 'pages' mode
        allLabel: 'All',
      }"
      :sort-options="{
        enabled: true,
        initialSortBy: {field: 'date', type: 'desc'}
      }"
      >
        <div slot="selected-row-actions">
          <button class="btn btn-success" :disabled="selectedTransactions.length == 0" @click="stageSelected">Stage Selected Transactions</button>
        </div>
      <template slot="table-row" slot-scope="props">
          <span v-if="props.column.field == 'amount'">
            {{currencyFormatter(props.row.amount)}}
          </span>
          <span v-else-if="props.column.field == 'account'" v-bind:id="'transaction-row-' + props.row.id">
              {{props.formattedRow[props.column.field]}}
          </span>
          <span v-else>
              {{props.formattedRow[props.column.field]}}
          </span>
      </template>
    </vue-good-table>
  </div>
</template>

<script>

import { VueGoodTable } from 'vue-good-table';

export default {
  data() {
    return {
      selectedTransactions: [],
      stagedTransactions: [],
      columns: [
        {
          label: 'Account',
          field: 'account_name',
          filterOptions: {
            enabled: true, // enable filter for this column
            placeholder: 'All Accounts', // placeholder for filter input
            filterDropdownItems: this.accounts.map(a => a.name)
          },
        },
        {
          label: 'Date',
          field: 'date',
          width: '115px',
          filterOptions: {
            enabled: true,
          }
        },
        {
          label: 'Payee',
          field: 'payee_name',
          filterOptions: {
            enabled: true, // enable filter for this column
          }
        },
        {
          label: 'Category',
          field: (t) => {
            if (t.category_name instanceof Array) {
              return t.category_name.join(", ");
            } else {
              return t.category_name;
            }
          },
          filterOptions: {
            enabled: true, // enable filter for this column
            placeholder: 'All Categories', // placeholder for filter input
            filterDropdownItems: this.categories.map(c => c.name) // dropdown (with selected values) instead of text input
          }
        },
        {
          label: 'Memo',
          field: 'memo',
          filterOptions: {
            enabled: true, // enable filter for this column
          }
        },
        {
          label: 'Amount',
          field: 'amount',
          width: '100px',
          tdClass: "text-right"
        }
      ]
    }
  },
  watch: {
    categories: function () {
      this.columns[3].filterOptions.filterDropdownItems = this.categories.map(c => c.name)
    },
    accounts: function() {
      this.columns[0].filterOptions.filterDropdownItems = this.accounts.map(a => a.name)
    },
  },
  props: ['transactions', "categories", "accounts", "currencyFormatter"],
  components: {VueGoodTable},
  methods: {
    selectionChanged(params) {
      this.selectedTransactions = params.selectedRows;
    },
    stageSelected() {
      this.stagedTransactions.push(...this.selectedTransactions)
      this.selectedTransactions = [];
      this.$emit('stagedTransactions', this.stagedTransactions)
    },
  }
}
</script>
