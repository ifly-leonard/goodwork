<template>
<div :class="{'hidden': (activeTab != 'offices')}">
  <!-- Create Office Form -->
  <div :class="{'hidden': !showCreateOfficeForm}">
    <div class="absolute inset-0 opacity-75 bg-gray-500 z-10"></div>
    <div id="create-project-form" class="fixed inset-x-0 w-1/3 z-10 bg-gray-200 mx-auto p-8 rounded">
      <p class="py-2">
        <input ref="focusInput" class="w-full shadow appearance-none border rounded py-2 px-3 text-gray-800"
          type="text" placeholder="Name" v-model="name">
        <span class="hidden"></span>
      </p>
      <p class="py-2">
        <input class="w-full shadow appearance-none border rounded py-2 px-3 text-gray-800"
          type="text" placeholder="Description" v-model="description">
        <span class="hidden"></span>
      </p>
      <div class="flex flex-row justify-between pt-8 bg-gray-200 rounded">
        <button @click="closeCreateOfficeModal" class="hover:font-bold text-red-400">Cancel</button>
        <button @click="createNewOffice" class="bg-teal-400 text-white font-medium hover:bg-teal-600 py-3 px-4 rounded">Create</button>
      </div>
    </div>
  </div>

  <div class="flex flex-row flex-wrap justify-center">
    <div class="bg-white shadow-md w-64 h-64 flex flex-col justify-center items-center text-center rounded m-4 cursor-pointer"
      @click="openCreateOfficeModal">
      <font-awesome-icon :icon="faPlus" class="text-gray-600 text-4xl"></font-awesome-icon>
      <span class="text-gray-800 pt-4">{{ 'Add a new office' | localize }}</span>
    </div>

    <office v-for="(office, index) in offices" :key="office.id" :index="index" :details="office"></office>
  </div>
</div>
</template>

<script>
import { mapState, mapActions } from 'vuex'
import office from './office'
import { faPlus } from '@fortawesome/free-solid-svg-icons'

export default {
  components: { office },
  props: {
    activeTab: {
      required: true,
      type: String
    }
  },
  data: () => ({
    showCreateOfficeForm: false,
    name: '',
    description: '',
    faPlus
  }),
  computed: mapState({
    offices: state => state.home.offices
  }),
  methods: {
    ...mapActions([
      'addOffice'
    ]),
    createNewOffice () {
      this.addOffice({name: this.name, description: this.description})
      this.closeCreateOfficeModal()
      this.name = ''
      this.description = ''
    },
    closeCreateOfficeModal () {
      this.showCreateOfficeForm = false
    },
    openCreateOfficeModal () {
      this.showCreateOfficeForm = true
      this.$nextTick(() => {
        this.$refs.focusInput.focus()
      })
    }
  }
}
</script>
