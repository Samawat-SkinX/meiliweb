<template>
  <section class="space-y-4">
    <h3
      class="inline-flex w-full items-center justify-between text-xl font-semibold">
      {{ t('titles.general') }}
    </h3>

    <Alert v-if="error" dismissable theme="danger" @close="error = null">
      <div class="flex items-start justify-between">
        <p class="grow">{{ error }}</p>
        <DocumentationLink
          v-if="error instanceof TaskError"
          :href="error.link"
          class="shrink-0 grow-0" />
      </div>
    </Alert>

    <form class="space-y-4" @reset.prevent="reset()" @submit.prevent="submit()">
      <Alert theme="warning" :title="t('notices.primaryKey.title')">
        {{ t('notices.primaryKey.text') }}
      </Alert>

      <UniqueId as="section" v-slot="{ id }" class="flex flex-col gap-1">
        <Label required :for="id">{{ t('labels.primaryKey') }}</Label>
        <input
          v-model="primaryKey"
          required
          autofocus
          autocomplete="off"
          type="text"
          class="form-input" />
      </UniqueId>

      <footer class="flex flex-col items-center justify-end sm:flex-row">
        <Buttons>
          <Button type="reset" :disabled="!modified || loading" />
          <Button type="submit" :disabled="!modified || loading" :loading />
        </Buttons>
      </footer>
    </form>

    <h3
      class="inline-flex w-full items-center justify-between text-xl font-semibold">
      {{ t('titles.renameIndex') }}
    </h3>

    <form @submit.prevent="renameIndex()" class="space-y-4">
      <UniqueId as="section" v-slot="{ id }" class="flex flex-col gap-1">
        <Label required :for="id">{{ t('labels.renameIndexUid') }}</Label>
        <input
          v-model="renameIndexUid"
          required
          autofocus
          autocomplete="off"
          type="text"
          class="form-input" />
      </UniqueId>

      <div class="flex justify-end">
        <Button
          type="submit"
          :loading="isRenaming"
          icon-on-right
          theme="primary"
          icon="heroicons:check">
          {{ t('actions.renameIndex') }}
        </Button>
      </div>
    </form>

    <h3
      class="inline-flex w-full items-center justify-between text-xl font-semibold">
      {{ t('titles.duplicateIndex') }}
    </h3>

    <form @submit.prevent="duplicateIndex()" class="space-y-4">
      <Alert theme="warning" :title="t('notices.duplicateIndex.title')">
        {{ t('notices.duplicateIndex.text') }}
      </Alert>
      <UniqueId as="section" v-slot="{ id }" class="flex flex-col gap-1">
        <Label required :for="id">{{ t('labels.duplicateIndexUid') }}</Label>
        <input
          v-model="duplicateIndexUid"
          required
          autofocus
          autocomplete="off"
          type="text"
          class="form-input" />
      </UniqueId>

      <div class="flex justify-end">
        <Button
          type="submit"
          :loading="isDuplicating"
          icon-on-right
          theme="primary"
          icon="heroicons:document-duplicate">
          {{ t('actions.duplicateIndex') }}
        </Button>
      </div>
    </form>

    <h3
      class="inline-flex w-full items-center justify-between text-xl font-semibold">
      {{ t('titles.dangerZone') }}
    </h3>

    <div class="flex items-center gap-2">
      <form @submit.prevent="dropIndex()">
        <Button type="submit" icon-on-right theme="primary" icon="oi:delete">
          {{ t('actions.dropIndex') }}
        </Button>
      </form>
      <form @submit.prevent="deleteAllDocuments()">
        <Button type="submit" icon-on-right theme="primary" icon="oi:delete">
          {{ t('actions.deleteAllDocuments') }}
        </Button>
      </form>
    </div>
  </section>
</template>

<script setup lang="ts">
import { resettableRef } from '~/utils'
import {
  TaskError,
  useFormSubmit,
  useIndexOperations,
  useMeiliClient,
  useTask,
} from '~/composables'
import { TOAST_FAILURE, TOAST_SUCCESS, useToasts } from '~/stores/toasts'
import Alert from '~/components/layout/Alert.vue'
import Buttons from '~/components/layout/forms/Buttons.vue'
import Button from '~/components/layout/forms/Button.vue'
import type { Task } from 'meilisearch'
import { promiseTimeout } from '@vueuse/core'
import DocumentationLink from '~/components/layout/DocumentationLink.vue'
import Label from '~/components/layout/forms/Label.vue'
import { navigateTo } from '#imports'

type Props = {
  indexUid: string
}
const props = defineProps<Props>()
const { t } = useI18n()
const meili = useMeiliClient()
const { confirm } = useConfirmationDialog()
const index = meili.index(props.indexUid)
const {
  value: primaryKey,
  reset,
  modified,
} = resettableRef((await index.fetchPrimaryKey()) as string)
const { loading, error, handle } = useFormSubmit({
  confirm: { text: t('confirmations.primaryKey.text') },
})
const { handle: handleIndexDrop } = useFormSubmit({
  confirm: {
    title: t('confirmations.dropIndex.title', { index: index.uid }),
    text: t('confirmations.dropIndex.text'),
    acceptButtonText: t('confirmations.dropIndex.acceptButtonText'),
  },
})
const { handle: handleDeleteAllDocuments } = useFormSubmit({
  confirm: {
    title: t('confirmations.deleteAllDocuments.title', { index: index.uid }),
    text: t('confirmations.deleteAllDocuments.text'),
    acceptButtonText: t('confirmations.deleteAllDocuments.acceptButtonText'),
  },
})
const processTask = useTask()
const { createToast } = useToasts()
const self = reactive({
  primaryKey,
  error,
  duplicateIndexUid: ref(`${props.indexUid}-copy`),
  renameIndexUid: ref(`${props.indexUid}-new`),
  isDuplicating: false,
  isRenaming: false,
})

const submit = async () => {
  const toast = createToast({
    ...TOAST_PLEASEWAIT(t),
    immediate: false,
    title: t('toasts.primaryKey.title'),
    text: t('toasts.primaryKey.pendingText'),
  })
  await handle(async () => {
    toast.spawn()
    await processTask(() => index.update({ primaryKey: self.primaryKey }), {
      onSuccess: async () => {
        toast.update({ ...TOAST_SUCCESS(t) })
        reset(self.primaryKey)
      },
      onCanceled: () =>
        toast.update({
          ...TOAST_FAILURE(t),
          text: t('toasts.texts.canceledTask'),
        }),
      onFailure: (task: Task) => {
        toast.update({
          ...TOAST_FAILURE(t),
          text: t('toasts.texts.failedTask'),
        })
        self.error = task.error as TaskError
      },
    })
    reset(self.primaryKey)
  })
}

const { duplicateIndex: doDuplicateIndex, renameIndex: doRenameIndex } =
  useIndexOperations()
const duplicateIndex = async () => {
  if (!(await confirm({ text: t('confirmations.duplicateIndex.text') }))) {
    return
  }
  try {
    const newIndexUid = await doDuplicateIndex(props.indexUid, {
      onStart: () => (self.isDuplicating = true),
      newIndexUid: self.duplicateIndexUid,
    })
    await promiseTimeout(1000)
    await navigateTo(`/indexes/${newIndexUid}/documents`)
  } finally {
    self.isDuplicating = false
  }
}
const renameIndex = async () => {
  if (!(await confirm({ text: t('confirmations.renameIndex.text') }))) {
    return
  }
  try {
    const newIndexUid = await doRenameIndex(props.indexUid, {
      onStart: () => (self.isRenaming = true),
      newIndexUid: self.renameIndexUid,
    })
    await promiseTimeout(1000)
    await navigateTo(`/indexes/${newIndexUid}/documents`)
  } finally {
    self.isRenaming = false
  }
}

const dropIndex = async () => {
  const toast = createToast({
    ...TOAST_PLEASEWAIT(t),
    immediate: false,
    title: t('toasts.dropIndex.title'),
  })
  await handleIndexDrop(async () => {
    toast.spawn()
    await processTask(() => meili.deleteIndex(index.uid), {
      onSuccess: async () => {
        toast.update({ ...TOAST_SUCCESS(t) })
        reset(self.primaryKey)
        await promiseTimeout(500)
        navigateTo('/indexes', { replace: true })
      },
      onCanceled: () =>
        toast.update({
          ...TOAST_FAILURE(t),
          text: t('toasts.texts.canceledTask'),
        }),
      onFailure: (task: Task) => {
        toast.update({
          ...TOAST_FAILURE(t),
          text: t('toasts.texts.failedTask'),
        })
        self.error = task.error as TaskError
      },
    })
  })
}

const deleteAllDocuments = async () => {
  const toast = createToast({
    ...TOAST_PLEASEWAIT(t),
    immediate: false,
    title: t('toasts.deleteAllDocuments.title'),
  })
  await handleDeleteAllDocuments(async () => {
    toast.spawn()
    await processTask(() => meili.index(index.uid).deleteAllDocuments(), {
      onSuccess: async () => {
        toast.update({ ...TOAST_SUCCESS(t) })
        reset(self.primaryKey)
        await promiseTimeout(500)
        navigateTo(`/indexes/${index.uid}/documents`, { replace: true })
      },
      onCanceled: () =>
        toast.update({
          ...TOAST_FAILURE(t),
          text: t('toasts.texts.canceledTask'),
        }),
      onFailure: (task: Task) => {
        toast.update({
          ...TOAST_FAILURE(t),
          text: t('toasts.texts.failedTask'),
        })
        self.error = task.error as TaskError
      },
    })
  })
}

useHead({
  title: `${t('titles.general')} - ${props.indexUid}`,
})

const { duplicateIndexUid, renameIndexUid, isDuplicating, isRenaming } =
  toRefs(self)
</script>

<i18n>
en:
  titles:
    general: General settings
    duplicateIndex: Duplicate index
    renameIndex: Rename index
    dangerZone: Danger Zone
  confirmations:
    primaryKey:
      text: Do you want to update your settings?
    duplicateIndex:
      text: Duplicate this index?
    renameIndex:
      text: Rename this index?
    dropIndex:
      title: "Drop `{index}`?"
      text: "CAUTION: This action cannot be reversed!"
      acceptButtonText: Yes, delete forever
    deleteAllDocuments:
      title: "Delete all documents from `{index}`?"
      text: "CAUTION: This action cannot be reversed!"
      acceptButtonText: Yes, delete forever
  toasts:
    primaryKey:
      title: Updating primary key...
    dropIndex:
      title: Deleting index...
    deleteAllDocuments:
      title: Deleting documents...
  labels:
    primaryKey: Primary Key
    duplicateIndexUid: New index name
    renameIndexUid: New index name
  notices:
    primaryKey:
      title: Warning
      text: You can freely update the primary key of an index as long as it contains no documents. To change the primary key of an index that already contains documents, you must first delete all documents in that index. You may then change the primary key and index your dataset again.
    duplicateIndex:
      title: Warning
      text: Your index will be duplicated by batches of documents. Ensure that your index is not being written to during the duplication process.
  actions:
    duplicateIndex: Duplicate index
    renameIndex: Rename index
    dropIndex: Delete index
    deleteAllDocuments: Delete all documents
</i18n>
