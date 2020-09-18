<template>
  <div id="app">
    <Nav :budget="budget" v-on:unset-budget="budgetId = null; budget = null; getBudgets();" />
    <div class="container">
      <!-- Display a loading message if loading -->
      <div v-if="loading" class="row">
        <h1 class="display-4">Loading...</h1>
      </div>
      <!-- Display an error if we got one -->
      <div v-if="error" class="row">
        <h1 class="display-4">Oops!</h1>
        <p class="lead">{{error}}</p>
        <button class="btn btn-primary" @click="resetToken">Try Again &gt;</button>
      </div>
      <!-- Otherwise show our app contents -->
      <div v-else>
        <!-- If we dont have a token ask the user to authorize with YNAB -->
        <div v-if="!ynab.token" class="row">
          <form>
            <div class="form-group">
              <h2>Hello!</h2>
              <p class="lead">If you would like to use this App, please authorize with YNAB!</p>
              <button
                @click="authorizeWithYNAB"
                class="btn btn-primary"
              >Authorize This App With YNAB &gt;</button>
            </div>
          </form>
        </div>

        <div v-if="completedOperation">
          <p v-if="completedOperation.error">An error occurred: {{ completedOperation.error}} </p>
          <p class="row">The following API calls were completed successfully:</p>
          <ul class="row list-group">
            <li class="list-group-item" :key="call" v-for="call in completedOperation.successCalls">{{ call }}</li>
          </ul>
          <button class="mt-2 btn btn-success" @click="resetOperation">Start New Operation</button>
        </div>

        <!-- Otherwise if we have a token, show the budget select -->
        <div v-if="!budgetId" class="row">
          <Budgets :budgets="budgets" :selectBudget="selectBudget" />
        </div>
        <!-- If a budget has been selected, display transactions from that budget -->
        <div v-else-if="!loading && !completedOperation&& budget">
          <div>
            <h3 class="row">Step 1: Select Transactions to Move</h3>
            <Transactions
              :transactions="transactions.filter(t => t.subtransactions === undefined)"
              :categories="categories.filter(c => !(c.hidden || c.deleted))"
              :accounts="accounts.filter(c => !(c.closed))"
              :currency-formatter="displayCurrency"
              v-on:stagedTransactions="updateStagedTransactions"
            />
          </div>
          <div v-if="stagedTransactions.length > 0">
            <h3 class="row">Step 2: Select Destination Category</h3>
            <p class="row">If you want to move transactions to a category that doesn't yet exist, please create it in the normal YNAB interface and then reload this page.</p>
            <div class="row form-inline">
              <label for="categorySelect">Category:</label>
              <v-select class="form-control col-md-4" id="categorySelect" :options="categories.filter(c => !(c.hidden || c.deleted)).map(c => ({label:c.name, id:c.id}))" v-model="selectedCategory"></v-select>
            </div>
          </div>
          <div v-if="selectedCategory">
            <h3 class="row">Step 3: Preview Move Operation</h3>
            <p class="row">Double check everything before pressing commit!</p>
            <vue-good-table
              class="row"
              :columns="columns"
              :rows="stagedTransactions"
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
                      :select-options="{
        disableSelectInfo: true
      }"
              :sort-options="{
                  enabled: true,
                  initialSortBy: {field: 'date', type: 'desc'}
                }">
                  <template slot="table-row" slot-scope="props">
            <span v-if="props.column.field == 'amount'">
              {{displayCurrency(props.row.amount)}}
            </span>
            <span v-else>
                {{props.formattedRow[props.column.field]}}
            </span>
        </template>
            </vue-good-table>
            <p class="row mt-3">
              These transactions will be moved to the {{selectedCategory.label}} category.
            </p>
            <p class="row">
              The budgeted amount will be adjusted from the old cateogries to {{selectedCategory.label}} in each month to cover the transaction amount.
            </p>
            <button
              class="btn row btn-primary"
              @click="commitOperation"
            >Move {{stagedTransactions.length}} Transactions</button>
          </div>
        </div>
      </div>
      <Footer />
    </div>  
  </div>
</template>

<script>
// Hooray! Here comes YNAB!
import * as ynab from 'ynab';

// Import our config for YNAB
import config from './config.json';

// Import Our Components to Compose Our App
import Nav from './components/Nav.vue';
import Footer from './components/Footer.vue';
import Budgets from './components/Budgets.vue';
import Transactions from './components/Transactions.vue';
import Vue from 'vue'
import vSelect from 'vue-select'
import 'vue-select/dist/vue-select.css';

Vue.component('v-select', vSelect);
// import the styles
import 'vue-good-table/dist/vue-good-table.css'
import { VueGoodTable } from 'vue-good-table';

export default {
  // The data to feed our templates
  data () {
    return {
      ynab: {
        clientId: config.clientId,
        redirectUri: config.redirectUri,
        token: null,
        api: null,
      },
      columns: [
        {label: 'Account',
          field: 'account_name',
        },
        {
          label: 'Date',
          field: 'date',
          width: '150px',
        },
        {
          label: 'Payee',
          field: 'payee_name',
 
        },
        {
          label: 'Old Category',
          field: (t) => {
            if (t.category_name instanceof Array) {
              return t.category_name.join(", ");
            } else {
              return t.category_name;
            }
          },
        },
        {
          label: 'New Category',
          field: () => this.selectedCategory.label,
        },
        {
          label: 'Amount to Move',
          field: 'amount',
          tdClass: "text-right"
          }
      ],
      loading: false,
      error: null,
      budgetId: 'default',
      budget: null,
      budgets: [],
      accounts: [],
      transactions: [],
      stagedTransactions: [],
      categories: [],
      selectedCategory: null,
      completedOperation: null,
    }
  },
  // When this component is created, check whether we need to get a token,
  // budgets or display the transactions
  created() {
    this.ynab.token = this.findYNABToken();
    if (this.ynab.token) {
      this.api = new ynab.api(this.ynab.token);
      if (!this.budgetId) {
        this.getBudgets();
      } else {
        this.selectBudget(this.budgetId);
      }
    }
  },
  components: {
    Nav,
    Footer,
    Budgets,
    Transactions,
    VueGoodTable
  },
  methods: {
    displayCurrency: function(milli) {
      let settings = this.budget.currency_format;
      let amt = Math.abs(ynab.utils.convertMilliUnitsToCurrencyAmount(milli, settings.decimal_digits)).toFixed(settings.decimal_digits);
      let amtSeperated = amt.toString().replace(/\B(?=(\d{3})+(?!\d))/g, settings.group_separator);
      let sign = milli < 0 ? "- " : "";
      if (settings.display_symbol) {
        return settings.symbol_first ? sign + settings.currency_symbol + amtSeperated : sign + amtSeperated + settings.currency_symbol;
      }
      return sign + amtSeperated;
    },
    // This uses the YNAB API to get a list of budgets
    getBudgets() {
      this.loading = true;
      this.error = null;
      this.api.budgets.getBudgets().then((res) => {
        this.budgets = res.data.budgets;
      }).catch((err) => {
        this.error = err.error.detail;
      }).finally(() => {
        this.loading = false;
      });
    },
    // This selects a budget and gets all the transactions in that budget
    selectBudget(id) {
      this.loading = true;
      this.error = null;
      this.budgetId = id;
      this.accounts = [];
      this.transactions = [];
      this.categories = [];
        this.api.budgets.getBudgetById(this.budgetId).then(res => {
        this.budget = res.data.budget;
        this.accounts = this.budget.accounts;
        this.categories = this.budget.categories;
        this.payees = this.budget.payees;
        this.payeeNames = this.payees.reduce((names, p) => {names[p.id] = p.name; return names;}, {});
        this.accountNames = this.accounts.reduce((names, a) => {names[a.id] = a.name; return names;}, {});
        this.categoryNames = this.categories.reduce((names, c) => {names[c.id] = c.name; return names;}, {});
        this.transactions = this.budget.transactions.map(t => {
          t.account_name = this.accountNames[t.account_id];
          t.category_name = this.categoryNames[t.category_id];
          t.payee_name = this.payeeNames[t.payee_id];
          return t;
        });
      let transactionDict = this.transactions.reduce((dict, t) => {dict[t.id] = t; return dict;}, {})
      this.budget.subtransactions.forEach(st => {
        let transaction = transactionDict[st.transaction_id];
        if (transaction.subtransactions === undefined) {
          transaction.subtransactions = [];
          transaction.category_name = [];
        }
        transaction.subtransactions.push(st);
        transaction.category_name.push(this.categoryNames[st.category_id]);
      })
      }).catch((err) => {
        console.error("ERROR", err)
        this.error = err.error.detail;
      }).finally(() => {
        this.loading = false;
      });
    },
    // This builds a URI to get an access token from YNAB
    // https://api.youneedabudget.com/#outh-applications
    authorizeWithYNAB(e) {
      e.preventDefault();
      const uri = `https://app.youneedabudget.com/oauth/authorize?client_id=${this.ynab.clientId}&redirect_uri=${this.ynab.redirectUri}&response_type=token`;
      location.replace(uri);
    },
    // Method to find a YNAB token
    // First it looks in the location.hash and then sessionStorage
    findYNABToken() {
      let token = null;
      const search = window.location.hash.substring(1).replace(/&/g, '","').replace(/=/g,'":"');
      if (search && search !== '') {
        // Try to get access_token from the hash returned by OAuth
        const params = JSON.parse('{"' + search + '"}', function(key, value) {
          return key === '' ? value : decodeURIComponent(value);
        });
        token = params.access_token;
        sessionStorage.setItem('ynab_access_token', token);
        window.location.hash = '';
      } else {
        // Otherwise try sessionStorage
        token = sessionStorage.getItem('ynab_access_token');
      }
      return token;
    },
    // Clear the token and start authorization over
    resetToken() {
      sessionStorage.removeItem('ynab_access_token');
      this.ynab.token = null;
      this.error = null;
    },
    resetOperation() {
      this.completedOperation = null;
      this.selectedCategory = null;
      this.stagedTransactions = [];
      selectBudget(this.budgetId)
    },
    updateStagedTransactions(stagedTransactions) {
      this.stagedTransactions = stagedTransactions;
    },
    commitOperation() {
      let operation = {
        transactions: this.stagedTransactions,
        category: this.selectedCategory.id,
        categoryName: this.selectedCategory.label,
        type: "moveAndRebudget",
        rebudgetAmounts: [],
        successCalls: [],
        error: null
      }
      // function to figure out how much money to move each month given
      // a list of transactions. Ignores transactions with subtransactions.
      function makeRebudgetamounts(transactions) {
        return transactions.reduce((rebudgetAmounts, t) => {
          if (t.subtransactions) {
            return rebudgetAmounts;
          }
          let month = t.date.slice(0,7) + '-01';
          if (rebudgetAmounts[month] === undefined) {
            rebudgetAmounts[month] = {};
            rebudgetAmounts[month][operation.category] = 0;
          }
          if (t.category_id === operation.category) {
            return;
          }
          if (rebudgetAmounts[month][t.category_id] === undefined) {
            rebudgetAmounts[month][t.category_id] = 0;
          }
          rebudgetAmounts[month][t.category_id] += t.amount;
          rebudgetAmounts[month][operation.category] -= t.amount;
          return rebudgetAmounts;
        }, {});
      }
      
      this.api.transactions.updateTransactions(this.budgetId, {
        transactions: operation.transactions.map(t => ({id: t.id, category_id: operation.category}))
      }).then(res => {
        let completed = res.data.transactions.filter(t => t.category_id === operation.category);
        operation.successCalls.push(`Categorized ${completed.length} transactions under ${operation.categoryName}.`)
        if (completed < operation.transactions.length) {
          operation.error = `${operation.transactions.length - completed.length} transactions were not changed due to an error.`
        }
        return completed;
      }).then((completed) => {
        let completedIds = completed.map(t => t.id)
        let allUpdates = [];
        operation.rebudgetAmounts = makeRebudgetamounts(operation.transactions.filter(t => completedIds.includes(t.id)))
        Object.keys(operation.rebudgetAmounts).forEach(month => {
          let monthUpdates = [];
          var totalDelta = 0;
          Object.keys(operation.rebudgetAmounts[month]).forEach(category_id => {
            let delta = operation.rebudgetAmounts[month][category_id];
            totalDelta += Math.abs(delta);
            let op = this.api.categories.getMonthCategoryById(this.budgetId, month, category_id).then(res => {
              let catMonth = res.data.category;
              let amt = catMonth.budgeted + delta;
              return this.api.categories.updateMonthCategory(this.budgetId, month, category_id, {category: {budgeted: amt}});
            })
            monthUpdates.push(op);
          });
          let monthOp = Promise.all(monthUpdates).then(() => {
            operation.successCalls.push(`Moved ${this.displayCurrency(totalDelta/2)} to ${operation.categoryName} for ${month}`)
          })
          allUpdates.push(monthOp);
        });
        return Promise.all(allUpdates);
      }).then(() => {
        this.completedOperation = operation;
      }).catch(err => {
        console.log("ERROR", err)
        operation.error = err.error.detail;r;
        this.completedOperation = operation;
      }).finally(() => {
        this.stagedTransactions = [];
        this.selectedCategory = null;
      })
    }
  }
}
</script>
