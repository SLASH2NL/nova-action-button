<template>
    <panel-item :field="field">
        <template v-slot:value>
            <button
                class="btn btn-default btn-primary flex items-center justify-center"
                @click="handleClick"
                :disabled="field.readonly"
                :style="`background-color: ${field.buttonColor} !important`"
            >
              <span>{{ buttonText }}</span>
              <component v-if="svg" :is="svg"></component>
            </button>

            <!-- Action Confirmation Modal -->
            <portal to="modals" transition="fade-transition">
                <component
                    v-if="confirmActionModalOpened"
                    class="text-left"
                    :is="field.action.component"
                    :working="working"
                    :selected-resources="selectedResources"
                    :resource-name="resourceName"
                    :action="selectedAction"
                    :errors="errors"
                    @confirm="executeAction"
                    @close="confirmActionModalOpened = false"
                />
            </portal>
        </template>
    </panel-item>
</template>

<script>
import { Errors, FormField, HandlesValidationErrors, InteractsWithResourceInformation } from 'laravel-nova'

export default {
    mixins: [FormField, HandlesValidationErrors, InteractsWithResourceInformation],

    props: {
      resource: String,
      resourceName: String,
      resourceId: Number,
      field: Object,
      queryString: {
        type: Object,
        default: () => ({
          currentSearch: '',
          encodedFilters: '',
          currentTrashed: '',
          viaResource: '',
          viaResourceId: '',
          viaRelationship: '',
        }),
      },
    },

    data: () => ({
      working: false,
      confirmActionModalOpened: false,
    }),

    methods: {
        /**
         * Determine what to do with the action button click.
         */
        handleClick() {
          if (this.withoutConfirmation) {
            this.executeAction();
          } else {
            this.openConfirmationModal();
          }
        },

        /**
         * Confirm with the user that they actually want to run the selected action.
         */
        openConfirmationModal() {
            this.confirmActionModalOpened = true
        },

        /**
         * Close the action confirmation modal.
         */
        closeConfirmationModal() {
            this.confirmActionModalOpened = false
            this.errors = new Errors()
        },

        /**
         * Execute the selected action.
         */
        executeAction() {
            this.working = true

            if (this.selectedResources.length == 0) {
                alert(this.__('Please select a resource to perform this action on.'))
                return
            }

            Nova.request({
                method: 'post',
                url: this.endpoint || `/nova-api/${this.resourceName}/action`,
                params: this.actionRequestQueryString,
                data: this.actionFormData(),
            })
            .then(response => {
                this.confirmActionModalOpened = false
                this.handleActionResponse(response.data)
                this.working = false
            })
            .catch(error => {
                this.working = false

                if (error.response.status == 422) {
                    this.errors = new Errors(error.response.data.errors)
                    Nova.error(this.__('There was a problem executing the action.'))
                }
            })
        },

        /**
         * Gather the action FormData for the given action.
         */
        actionFormData() {
            return _.tap(new FormData(), formData => {
                formData.append('resources', this.selectedResources)

                _.each(this.selectedAction.fields, field => {
                    field.fill(formData)
                })
            })
        },

        /**
         * Handle the action response. Typically either a message, download or a redirect.
         */
        handleActionResponse(data) {
            try {
              this.$parent.$parent.$children[2].$emit('actionExecuted')
            } catch (e) {
              // Somehow didn't work. We continue so that the response is processed anyway.
            }
            if (data.message) {
                Nova.$emit('action-executed')
                Nova.success(data.message)
            } else if (data.deleted) {
                Nova.$emit('action-executed')
            } else if (data.danger) {
                Nova.$emit('action-executed')
                Nova.error(data.danger)
            } else if (data.download) {
                let link = document.createElement('a')
                link.href = data.download
                link.download = data.name
                document.body.appendChild(link)
                link.click()
                document.body.removeChild(link)
            } else if (data.redirect) {
                window.location = data.redirect
            } else if (data.push) {
                this.$router.push(data.push)
            } else if (data.openInNewTab) {
                window.open(data.openInNewTab, '_blank')
            } else {
                Nova.$emit('action-executed')
                Nova.success(this.__('The action ran successfully!'))
            }
        },
    },

    computed: {
        selectedResources() {
            return this.field.resourceId;
        },

        selectedAction() {
            return this.field.action;
        },

        /**
         * Get the query string for an action request.
         */
        actionRequestQueryString() {
            return {
            action: this.selectedAction.uriKey,
            search: this.queryString.currentSearch,
            filters: this.queryString.encodedFilters,
            trashed: this.queryString.currentTrashed,
            viaResource: this.queryString.viaResource,
            viaResourceId: this.queryString.viaResourceId,
            viaRelationship: this.queryString.viaRelationship,
            }
        },

        buttonText() {
            return this.field.text || this.__('Run');
        },

        withoutConfirmation() {
          return this.field.withoutConfirmation || false;
        },

        svg() {
          return this.field.svg || false;
        }
    }
}
</script>
