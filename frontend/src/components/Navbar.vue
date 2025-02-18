<template>
  <nav class="bg-white border-b">
    <div class="mx-auto py-2 px-5 md:pl-3 md:pr-8 h-16 md:h-12 z-10 flex items-center justify-between">
      <div class="flex items-center">
        <router-link to="/" class="hidden md:block">
          <FrappeDriveLogo class="h-4" />
        </router-link>
        <div class="flex items-center md:hidden">
          <button
            class="mr-5 inline-flex items-center justify-center text-gray-700 rounded-md focus:outline-none focus:shadow-outline-gray"
            @click="$emit('toggleMobileSidebar')">
            <FeatherIcon v-if="!mobileSidebarIsOpen" name="menu" class="w-6 h-6" />
            <FeatherIcon v-else name="x" class="w-6 h-6" />
          </button>
          <router-link to="/">
            <FrappeLogo class="h-6 w-auto" />
          </router-link>
        </div>
      </div>

      <div class="flex items-center">
        <Input iconLeft="search" type="text" class="cursor-pointer" placeholder="Search" readonly
          @click="showSearchDialog=true" />
        <Button class="ml-4 md:ml-6" appearance="minimal" icon="bell"></Button>
        <div class="relative ml-3">
          <Dropdown :options="dropdownItems" placement="right">
            <button
              class="flex items-center max-w-xs text-sm text-white rounded-full focus:outline-none focus:shadow-solid"
              id="user-menu" aria-label="User menu" aria-haspopup="true">
              <div class="flex items-center gap-4">
                <Avatar :label="fullName" :imageURL="imageURL" />
              </div>
            </button>
          </Dropdown>
        </div>
      </div>
    </div>
  </nav>
  <SearchDialog v-model="showSearchDialog" @openEntity="(entity) => openEntity(entity)" />
  <FilePreview v-if="showPreview" @hide="hidePreview" :previewEntity="previewEntity" />
</template>
<script>
import { Avatar, Dropdown, FeatherIcon, Input, Button } from 'frappe-ui'
import FrappeDriveLogo from '@/components/FrappeDriveLogo.vue'
import SearchDialog from '@/components/SearchDialog.vue'
import FilePreview from '@/components/FilePreview.vue'
import FrappeLogo from '@/components/FrappeLogo.vue'

export default {
  name: 'Navbar',
  components: {
    FilePreview,
    FrappeDriveLogo,
    FrappeLogo,
    SearchDialog,
    Avatar,
    Dropdown,
    FeatherIcon,
    Input,
    Button
  },
  props: {
    mobileSidebarIsOpen: {
      type: Boolean,
      required: true,
    },
  },
  emits: ['toggleMobileSidebar'],
  data() {
    return {
      previewEntity: null,
      showPreview: false,
      dropdownItems: [
        {
          label: 'Log out',
          handler: () => this.$store.dispatch('logout'),
        },
      ],
      showSearchDialog: false
    }
  },
  computed: {
    fullName() {
      return this.$store.state.user.fullName
    },
    imageURL() {
      return this.$store.state.user.imageURL
    },
  },
  methods: {
    openEntity(entity) {
      if (entity.is_group) {
        const name = entity?.read ? 'SharedFolder' : 'Folder'
        this.$router.push({
          name: name,
          params: { entityName: entity.name },
        })
      } else {
        this.previewEntity = entity
        this.showPreview = true
      }
    },
    hidePreview() {
      this.showPreview = false
      this.previewEntity = null
    },
  },
}
</script>
