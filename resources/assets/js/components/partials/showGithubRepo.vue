<template>
<div>
  <div class="absolute container mx-auto w-1/3 bg-white rounded shadow-lg z-10 mb-12 pb-4" style="top: 10vh;left: 0;right: 0;">
    <div class="text-gray-600 text-center py-4 flex flex-row justify-around items-center">
      <span></span>
      <span class="text-2xl">Your Github Repositories</span>
      <span @click="closeModal">
        <font-awesome-icon :icon="faTimes" class="cursor-pointer text-red-200 text-base"></font-awesome-icon>
      </span>
    </div>
    <ul class="pl-0">
      <li v-for="(repository, index) in repositories" class="list-reset flex flex-row justify-between items-center py-4 px-8 hover:bg-blue-600">
        <div class="text-gray-800 text-xl">
          {{ repository.name }}
        </div>
        <div>
          <button @click="connectGithubRepo(repository.id, repository.name)" class="text-teal-600 font-semibold text-sm border border-teal-400  p-2 rounded">Connect</button>
        </div>
      </li>
    </ul>
    <div v-if="accessTokenNotSet" class="p-8 text-gray-800">
      <div class="bg-red-100 border-l border-red-400 p-4 rounded">
        <div class="text-lg text-red-800">Access Token is not set.</div>
        <div class="text-sm">Please set your Github Access Token in intregation settings on Admin page.</div>
      </div>
    </div>
  </div>

  <div @click="closeModal" class="h-screen w-screen fixed inset-0 bg-gray-900 opacity-25"></div>
</div>
</template>

<script>
import { mapActions } from 'vuex'
import { faTimes } from '@fortawesome/free-solid-svg-icons'

export default {
  props: {
    entityType: {
      required: true,
      type: String
    },
    entityId: {
      required: true,
      type: Number
    }
  },

  data: () => ({
    repositories: [],
    accessTokenNotSet: false,
    faTimes
  }),

  created () {
    axios.get('/services/github/repos')
      .then((response) => {
        this.repositories = response.data.repos.data.viewer.repositories.nodes
      })
      .catch(() => {
        this.accessTokenNotSet = true
      })
  },

  methods: {
    ...mapActions([
      'closeComponent',
      'showNotification'
    ]),
    closeModal () {
      this.closeComponent()
    },
    connectGithubRepo (id, name) {
      axios.post('/services/github/connected-repos', {
        github_repo_id: id,
        repo_name: name,
        entity_type: this.entityType,
        entity_id: this.entityId
      })
        .then((response) => {
          this.showNotification({type: response.data.status, message: response.data.message})
        })
        .catch((error) => {
          this.showNotification({type: error.response.data.status, message: error.response.data.message})
        })
    }
  }
}
</script>
