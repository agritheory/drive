<template>
  <Dialog :options="{ title: 'Sharing options' }" v-model="open">
    <template #body-content>
      <div class="text-left min-w-[16rem]" ref="dialogContent">
        <div class="flex mt-5 text-sm font-medium text-gray-600">
          <FeatherIcon name="user-plus" :strokeWidth="2" class="h-4 pr-1" />Shared with
        </div>
        <div v-for="user in $resources.sharedWith.data" :key="user.user"
          class="mt-3 flex justify-between items-center text-base font-medium antialiased">
          <div class="flex w-full gap-2 items-center">
            <div class="overflow-hidden rounded-full h-7 w-7">
              <img v-if="user.user_image" :src="user.user_image" class="object-cover rounded-full h-7 w-7" />
              <div v-else
                class="flex items-center justify-center w-full h-full text-base text-gray-600 uppercase bg-gray-200">
                {{ user.full_name[0] }}
              </div>
            </div>
            <div class="max-w-[70%] truncate">{{ user.user }}</div>
          </div>
          <Dropdown placement="right" :options="
            [
              { label: 'Viewer' },
              { label: 'Editor' },
              { label: 'Delete' },
            ].map((option) => ({
              ...option,
              handler: () => {
                user.loading = true
                $resources.share
                  .submit(
                    Object.assign(
                      {
                        method:
                          option.label === 'Delete' ? 'unshare' : 'share',
                        entity_name: entityName,
                        user: user.user,
                      },
                      option.label != 'Delete' && {
                        write: option.label === 'Editor' ? 1 : 0,
                      }
                    )
                  )
                  .then(() => {
                    user.loading = false
                  })
              },
            }))
          ">
            <Button iconRight="chevron-down" :loading="user.loading"
              class="text-sm w-24 focus:ring-0 focus:ring-offset-0" appearance="minimal">{{ user.write ? 'Editor' :
              'Viewer'
              }}</Button>
          </Dropdown>
        </div>
        <Button v-if="!showUserSearch" class="mt-3 focus:ring-0 focus:ring-offset-0" icon="plus"
          @click="showUserSearch = true" />
        <div v-if="showUserSearch" class="flex items-start mt-5 gap-2 w-full relative">
          <UserSearch class="flex-1 invisible" />
          <UserSearch v-model="searchQuery" @submit="
            (user) =>
              $resources.share.submit({
                method: 'share',
                entity_name: entityName,
                user: user,
                write: 0,
                share: 1,
              })
          " class="flex-1 absolute w-[calc(100%_-_2.25rem)]" />
          <Button class="focus:ring-0 focus:ring-offset-0 absolute left-[calc(100%_-_1.75rem)]" icon="plus"
            appearance="primary" @click="$resources.share.fetch()" />
        </div>
        <ErrorMessage v-if="$resources.share.error" class="mt-2" :message="errorMessage" />
        <hr class="mt-5" />
        <div class="flex mt-5 text-base font-medium text-gray-600">
          <FeatherIcon name="unlock" :strokeWidth="2" class="h-4 pr-1" />General access
        </div>
        <div class="flex mt-3">
          <Dropdown placement="left" :options="
            [
              { label: 'Restricted', read: 0, write: 0 },
              { label: 'Open', read: 1, write: 0 },
            ].map((option) => ({
              ...option,
              handler: () => {
                if (generalAccess ? !option.read : option.read) {
                  generalAccessLoading = true
                  $resources.updateAccess.submit({
                    method: 'set_general_access',
                    entity_name: entityName,
                    new_access: { read: option.read, write: option.write }
                  })
                    .then(() => {
                      generalAccessLoading = false
                    })
                }
              },
            }))
          ">
            <Button iconRight="chevron-down" class="text-sm font-medium w-28 focus:ring-0 focus:ring-offset-0"
              :loading="generalAccessLoading">
              {{ generalAccess ? "Open" : "Restricted" }}
            </Button>
          </Dropdown>
          <Dropdown v-if="generalAccess" class="ml-auto" :options="
            [
              { label: 'For viewing', read: 1, write: 0 },
              { label: 'For editing', read: 1, write: 1 },
            ].map((option) => ({
              ...option,
              handler: () => {
                if (option.write !== generalAccess.write) {
                  generalAccessLoading = true
                  $resources.updateAccess.submit({
                    method: 'set_general_access',
                    entity_name: entityName,
                    new_access: { read: option.read, write: option.write }
                  })
                    .then(() => {
                      generalAccessLoading = false
                    })
                }
              },
            }))
          ">
            <Button iconRight="chevron-down" class="text-sm w-30 focus:ring-0 focus:ring-offset-0" appearance="minimal"
              :loading="generalAccessLoading">
              {{ generalAccess.write ? "For editing" : "For viewing" }}
            </Button>

          </Dropdown>
        </div>
        <p class="mt-2 text-xs text-gray-600">{{ accessMessage }}</p>
        <div class="flex mt-5">
          <Button icon-left="link" appearance="white" @click="getLink">Get link</Button>
          <Button appearance="primary" class="ml-auto w-24" @click="open = false">Done</Button>
        </div>
        <Alert :title="alertMessage" class="mt-5" v-if="showAlert"></Alert>
      </div>
    </template>
  </Dialog>
</template>
<script>
import { Dialog, ErrorMessage, FeatherIcon, Button, Dropdown, Alert } from 'frappe-ui'
import UserSearch from '@/components/UserSearch.vue'

export default {
  name: 'ShareDialog',
  components: {
    Dialog,
    ErrorMessage,
    FeatherIcon,
    Button,
    UserSearch,
    Dropdown,
    Alert
  },
  props: {
    modelValue: {
      type: Boolean,
      required: true,
    },
    entityName: {
      type: String,
      required: true,
    },
    isFolder: {
      type: Boolean,
      required: true,
    },
  },
  emits: ['update:modelValue', 'success'],
  data() {
    return {
      generalAccess: {},
      generalAccessLoading: false,
      showUserSearch: false,
      searchQuery: '',
      errorMessage: '',
      showAlert: false,
      alertMessage: ""
    }
  },
  computed: {
    accessMessage() {
      if (this.generalAccess)
        return this.generalAccess.write ? "Anybody with the link who is signed in can edit" : "Anybody with the link can view"
      return "Only shared with individuals can view or edit"
    },
    open: {
      get() {
        return this.modelValue
      },
      set(value) {
        this.$emit('update:modelValue', value)
        if (!value) {
          this.errorMessage = ''
        }
      },
    },
  },
  methods: {
    async getLink() {
      this.showAlert = false
      const link = this.isFolder ?
        `${window.location.origin}/drive/shared/folder/${this.entityName}` :
        `${window.location.origin}/drive/file/${this.entityName}`
      try {
        await navigator.clipboard.writeText(link);
        this.alertMessage = "Link copied successfully"
        this.showAlert = true
      } catch ($e) {
        this.alertMessage = "Some error occurred while copying the link"
        this.showAlert = true
      }
    }
  },
  mounted() {
    const targetElement = this.$refs.dialogContent?.closest('.overflow-hidden')
    if (targetElement) {
      targetElement.classList.remove('overflow-hidden')
      targetElement.classList.add('self-center')
      targetElement.childNodes.forEach((node) => {
        if (node.nodeType === Node.ELEMENT_NODE)
          node.classList.add('rounded-lg')
      })
    }
  },
  resources: {
    sharedWith() {
      return {
        method: 'drive.api.permissions.get_shared_with_list',
        params: {
          entity_name: this.entityName,
        },
        auto: true
      }
    },
    generalAccess() {
      return {
        method: 'drive.api.permissions.get_general_access',
        params: { entity_name: this.entityName, },
        onSuccess(data) {
          this.generalAccess = data
        },
        auto: true
      }
    },
    share() {
      return {
        method: 'drive.api.files.call_controller_method',
        params: {
          method: 'share',
          entity_name: this.entityName,
          user: this.searchQuery,
          write: 0,
        },
        validate(params) {
          if (!params?.user) {
            return 'No user was specified'
          }
        },
        onSuccess() {
          this.$resources.share.error = null
          this.$resources.sharedWith.fetch()
          this.showUserSearch = false
          this.searchQuery = ''
        },
        onError(error) {
          if (error.messages) {
            this.errorMessage = error.messages.join('\n')
          } else {
            this.errorMessage = error.message
          }
        },
      }
    },
    updateAccess() {
      return {
        method: 'drive.api.files.call_controller_method',
        onSuccess() {
          this.$resources.generalAccess.fetch()
        },
        onError(error) {
          if (error.messages) {
            console.log(error.messages);
          }
        },
      }
    },
  },
}
</script>
