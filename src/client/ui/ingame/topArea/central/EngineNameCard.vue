<template>
    <!-- Keys typed here are the name's, not the table's commands ( Space passes ) ; pointers too :
         the table under the central box takes them otherwise -->
    <div
        class="engine-name-card"
        @pointerdown.stop
        @pointerup.stop
        @pointermove.stop
        @keydown.stop
        @keyup.stop
    >
        <input
            ref="input"
            v-model="typed"
            type="text"
            :placeholder="failed ? 'The cards cannot be fetched' : 'Name a card'"
            :disabled="failed"
            @keydown.enter="name(matches[0])"
        />
        <div class="engine-choices-buttons">
            <button
                v-for="card in matches"
                :key="card.id"
                class="game-button small"
                @click="name(card)"
            >
                {{ card.name }}
            </button>
        </div>
    </div>
</template>

<script setup lang="ts">
/**
 * A play naming any library card ( Precision, Sibyl's Tongue ) : one option, answered with the card
 * named, completed among the library cards the engine plays.
 */
import { computed, onMounted, ref, useTemplateRef } from 'vue'
import { engineLibraryCards } from '@/client/engine/host.ts'
import { chooseEngineOption } from '@/client/engine/driver.ts'
import * as logging from '@/client/logging.ts'
import type { OptionView } from '@/client/engine/api.ts'

const props = defineProps<{ option: OptionView }>()

const MAX_MATCHES = 8

const cards = ref<{ id: number; name: string; key: string }[]>([])
const failed = ref(false)
const typed = ref('')
const input = useTemplateRef('input')

onMounted(async () => {
    input.value?.focus()
    try {
        cards.value = (await engineLibraryCards()).map(card => ({
            ...card,
            key: simple(card.name),
        }))
    } catch (error) {
        failed.value = true
        logging.captureException(error)
    }
})

// A name as typed : no case, no accents, no punctuation
function simple(name: string): string {
    return name
        .normalize('NFD')
        .replace(/[̀-ͯ]/g, '')
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, ' ')
        .trim()
}

// The names starting with what is typed first, then those containing it
const matches = computed(() => {
    const key = simple(typed.value)
    if (!key) {
        return []
    }
    const starting = cards.value.filter(card => card.key.startsWith(key))
    const containing = cards.value.filter(
        card => !card.key.startsWith(key) && card.key.includes(key),
    )
    return [...starting, ...containing].slice(0, MAX_MATCHES)
})

function name(card: { id: number } | undefined) {
    if (card) {
        chooseEngineOption(props.option.index, card.id)
    }
}
</script>

<style lang="scss">
.engine-name-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 6px;

    input {
        width: 18rem;
        font-size: 15px;
    }
}
</style>
