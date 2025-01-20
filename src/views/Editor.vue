<script setup lang="ts">
import { useRoute, useRouter, onBeforeRouteLeave } from 'vue-router'
import { useDraftsStore } from '@/logics/drafts'
import { genClient } from '@/logics/auth'
import type { Draft } from '@/logics/drafts'
import { ref, computed, watch } from 'vue'

const supabase = genClient()

const route = useRoute()
const router = useRouter()
const title = route.params.title as unknown as string
const draftsStore = useDraftsStore()
// text is to be shown in the textarea:
//   # TITLE
//   // empty line
//   <MAIN CONTENT>
const genText = (c_: string) => `# ${title}\n\n${c_}`
const text = ref(genText(draftsStore.getDraft(title).content))
// content is the main content of the draft
const content = computed(
  () => text.value.split('\n').slice(1).join('\n').trim() + '\n'
)

const savingLoading = ref(false)
const refreshLoading = ref(false)

const refresh = async () => {
  refreshLoading.value = true
  await draftsStore.refreshDraft(supabase)
  text.value = genText(draftsStore.getDraft(title).content)
  refreshLoading.value = false
}
const save = async (leaving: boolean) => {
  let newTitle = text.value.split('\n')[0]
  if (newTitle[0] === '#') newTitle = newTitle.slice(1)
  newTitle = newTitle.trim()
  if (
    draftsStore.getDraft(title).content === content.value &&
    (newTitle === title ||
      draftsStore.drafts.findIndex((d) => d.title === title) === -1)
  )
    return
  savingLoading.value = true
  await draftsStore.updateDraft(
    supabase,
    {
      title: newTitle,
      content: content.value
    },
    title
  )
  savingLoading.value = false
  if (!leaving && newTitle !== title) router.push(`/editor/${newTitle}`)
}
const copy = () =>
  navigator.clipboard.write(
    new ClipboardItem({
      "text/plain": new Blob(
        [text.value.trim() + '🏝️'],
        { type: "text/plain" }
      )
    })
  )

const paraNum = computed(
  () => content.value.split('\n').filter((p) => p.length > 0).length
)
const charNumTotal = computed(() => content.value.length)
const charNumNoPunc = computed(() => {
  const punc = '.,?!;:()[]{}"\'$。，？！；：（）【】｛｝—、“”’‘¥ \t\n\r'.split(
    ''
  )
  return content.value.split('').filter((c) => !punc.includes(c)).length
})

onBeforeRouteLeave(async () => await save(true))
</script>

<template>
  <div class="flex flex-col">
    <div class="flex items-center">
      <button @click="router.push('/home')" class="flex hover-up">
        <span class="i-charm:cards"></span>
      </button>
      <button @click="save(false)" class="flex hover-up">
        <span
          class="i-charm:floppy-disk"
          :class="savingLoading ? 'animate-bounce' : ''"
        ></span>
      </button>
      <button @click="refresh" class="flex hover-up">
        <span
          class="i-charm:refresh"
          :class="refreshLoading ? 'animate-spin' : ''"
        ></span>
      </button>
      <button @click="copy" class="flex hover-up">
        <span
          class="i-charm:copy"
        ></span>
      </button>
    </div>
    <div class="mx-6px my-7px">
      {{ paraNum }} / {{ charNumTotal }} / {{ charNumNoPunc }}
    </div>
    <textarea v-model="text" class="h-76vh w-full"></textarea>
  </div>
</template>

<style scoped>
button {
  padding: 3px;
}
button span {
  font-size: 1.1em;
}
</style>
