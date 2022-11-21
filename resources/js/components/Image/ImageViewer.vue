<template>
    <div v-if="hasValue" class="mb-6">
        <div class="relative" style="width: fit-content;">
            <template v-if="shouldShowLoader">
                <image-loader :src="field.previewUrl" class="max-w-xs" @missing="(value) => missing = value" />
            </template>

            <button
                v-if="shouldShowRemoveButton"
                @click="confirmRemoval"
                type="button"
                class="rounded-full shadow bg-white dark:bg-gray-800 text-center flex items-center justify-center h-[20px] w-[21px] absolute z-20 top-[-10px] right-[-9px] v-popper--has-tooltip" dusk="profile_photo_path-delete-link"
            >
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" width="20" height="20" class="inline-block text-gray-800 dark:text-gray-200" role="presentation"><path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd"></path></svg>
            </button>
        </div>

        <template v-if="field.value && !field.previewUrl">
            <card class="flex item-center relative border border-lg border-50 overflow-hidden p-4">
                {{ field.value }}

<!--                <DangerButton-->
<!--                    type="delete"-->
<!--                    class="ml-auto"-->
<!--                    v-if="shouldShowRemoveButton"-->
<!--                    @click="confirmRemoval"-->
<!--                />-->
            </card>
        </template>

        <p
            v-if="field.previewUrl"
            class="mt-3 flex items-center text-sm"
        >
<!--            <DangerButton-->
<!--                type="delete"-->
<!--                v-if="shouldShowRemoveButton"-->
<!--                @click="confirmRemoval"-->
<!--            >-->
<!--                <span class="class ml-2 mt-1">-->
<!--                    {{__('Delete')}}-->
<!--                </span>-->
<!--            </DangerButton>-->
        </p>

        <portal to="modals">
            <transition name="fade">
                <confirm-upload-removal-modal
                    v-if="removeModalOpen"
                    @confirm="removeFile"
                    @close="closeRemoveModal"
                />
            </transition>
        </portal>

        <!-- Upload Removal Modal -->
        <ConfirmUploadRemovalModal
            :show="removeModalOpen"
            @confirm="removeUploadedFile"
            @close="closeRemoveModal"
        />
    </div>
</template>

<script>
import { Errors } from 'laravel-nova'

import ImageLoader from './ImageLoader'

export default {
    components: { ImageLoader },

    props: ['field', 'resourceId', 'resourceName', 'relatedResourceId', 'relatedResourceName', 'viaRelationship'],

    data: () => ({
        removeModalOpen: false,
        missing: false,
        deleted: false,
    }),

    methods: {
        /**
         * Confirm removal of the linked file
         */
        confirmRemoval() {
            this.removeModalOpen = true
        },

        /**
         * Close the upload removal modal
         */
        closeRemoveModal() {
            this.removeModalOpen = false
        },

        /**
         * Remove the linked file from storage
         */
        async removeFile() {
            this.uploadErrors = new Errors()

            const {
                resourceName,
                resourceId,
                relatedResourceName,
                relatedResourceId,
                viaRelationship,
            } = this
            const attribute = this.field.attribute

            const uri = this.viaRelationship
                ? `/nova-api/${resourceName}/${resourceId}/${relatedResourceName}/${relatedResourceId}/field/${attribute}?viaRelationship=${viaRelationship}`
                : `/nova-api/${resourceName}/${resourceId}/field/${attribute}`

            try {
                await Nova.request().delete(uri)
                this.closeRemoveModal()
                this.deleted = true
                this.$emit('image-deleted')
            } catch (error) {
                this.closeRemoveModal()

                if (error.response.status == 422) {
                    this.uploadErrors = new Errors(error.response.data.errors)
                }
            }
        },

        async removeUploadedFile() {
            this.uploadErrors = new Errors()

            const {
                resourceName,
                resourceId,
                relatedResourceName,
                relatedResourceId,
                viaRelationship,
            } = this
            const attribute = this.field.attribute

            const uri =
                this.viaRelationship &&
                this.relatedResourceName &&
                this.relatedResourceId
                    ? `/nova-api/${resourceName}/${resourceId}/${relatedResourceName}/${relatedResourceId}/field/${attribute}?viaRelationship=${viaRelationship}`
                    : `/nova-api/${resourceName}/${resourceId}/field/${attribute}`

            try {
                await Nova.request().delete(uri)
                this.closeRemoveModal()
                this.deleted = true
                this.$emit('file-deleted')
                Nova.success(this.__('The file was deleted!'))
            } catch (error) {
                this.closeRemoveModal()

                if (error.response?.status === 422) {
                    this.uploadErrors = new Errors(error.response.data.errors)
                }
            }
        },
    },

    computed: {
        /**
         * Determine whether the field has a value
         */
        hasValue() {
            return (
                Boolean(this.field.value || this.field.previewUrl) &&
                !Boolean(this.deleted) &&
                !Boolean(this.missing)
            )
        },

        /**
         * Determine whether the field should show the loader component
         */
        shouldShowLoader() {
            return !Boolean(this.deleted) && Boolean(this.field.thumbnailUrl)
        },

        /**
         * Determine whether the field should show the remove button
         */
        shouldShowRemoveButton() {
            return Boolean(this.field.deletable)
        },
    },
}
</script>
