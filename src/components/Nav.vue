<template>
  <div class="navbar navbar-dark bg-primary d-flex flex-column flex-md-row align-items-center p-3 px-md-4 mb-3 border-bottom box-shadow">
    <div class="container">
      <h1 class="my-0 mr-md-auto font-weight-normal navbar-brand">YNAB Transaction Mover</h1>
      <div v-if="budget">
        <h2 class="font-weight-normal navbar-brand">{{ budget.name }}</h2>
        <button class="btn btn-info float-right" @click="$emit('unset-budget')">&lt; Select Another Budget</button>
      </div>
    </div>
  </div>
</template>


<script>
export default {
  props: ['budget']
}
</script>
