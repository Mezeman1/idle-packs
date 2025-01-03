<template>
    <div v-if="show" class="fixed inset-0 flex items-start justify-center z-50">
        <!-- Backdrop -->
        <div class="absolute inset-0 bg-black bg-opacity-50"></div>

        <!-- Modal -->
        <div class="relative bg-white rounded-lg shadow-xl max-w-md w-full mx-4 mt-8">
            <!-- Fixed Header -->
            <div class="sticky top-0 bg-white rounded-t-lg border-b p-4 z-10">
                <div class="flex justify-between items-center mb-4">
                    <h2 class="text-2xl font-bold">Opening {{ packName }}</h2>
                    <div v-if="remainingItems === 0" class="flex gap-2">
                        <button v-if="hasMorePacks" @click="openAnother"
                            class="px-4 py-2 bg-blue-500 text-white text-sm rounded-lg hover:bg-blue-600">
                            Open Another
                        </button>
                        <button @click="close"
                            class="px-4 py-2 bg-green-500 text-white text-sm rounded-lg hover:bg-green-600">
                            Done
                        </button>
                    </div>
                    <div v-else class="flex gap-2">
                        <button @click="revealNext"
                            class="px-4 py-2 bg-blue-500 text-white text-sm rounded-lg hover:bg-blue-600">
                            Next
                        </button>
                        <button @click="revealAll"
                            class="px-4 py-2 bg-blue-600 text-white text-sm rounded-lg hover:bg-blue-700">
                            All
                        </button>
                    </div>
                </div>

                <!-- Progress info -->
                <div class="flex justify-between items-center text-sm">
                    <div class="flex items-center gap-2">
                        <span v-if="remainingItems > 0">
                            Remaining Items: {{ remainingItems }}
                        </span>
                        <span v-if="remainingPacks > 0" class="text-blue-600">
                            ({{ remainingPacks }} more packs)
                        </span>
                    </div>
                    <div v-if="revealedItems.length > 0" class="flex items-center gap-2">
                        <span>Total Value:</span>
                        <span :class="{ 'text-green-600': isProfit, 'text-red-600': !isProfit }">
                            {{ formatNumber(totalValue) }}
                            <span class="ml-1">
                                ({{ isProfit ? '+' : '' }}{{ formatNumber(profit) }})
                            </span>
                        </span>
                    </div>
                </div>
            </div>

            <!-- Scrollable Content -->
            <div class="p-4">
                <!-- Revealed items -->
                <div class="space-y-3 max-h-[60vh] overflow-y-auto pr-2">
                    <div v-for="item in stackedItems" :key="item.id"
                        class="border p-3 rounded-lg flex items-center justify-between animate-fade-in" :class="{
                            'border-gray-300': item.rarity === 'common',
                            'border-green-400': item.rarity === 'uncommon',
                            'border-blue-400': item.rarity === 'rare',
                            'border-purple-400': item.rarity === 'epic',
                            'border-yellow-400 animate-pulse': item.rarity === 'legendary',
                        }">
                        <div>
                            <h3 class="font-bold">{{ item.name }}</h3>
                            <div class="flex items-center gap-2">
                                <p class="text-sm capitalize" :class="{
                                    'text-gray-600': item.rarity === 'common',
                                    'text-green-600': item.rarity === 'uncommon',
                                    'text-blue-600': item.rarity === 'rare',
                                    'text-purple-600': item.rarity === 'epic',
                                    'text-yellow-600': item.rarity === 'legendary',
                                }">
                                    {{ item.rarity }}
                                </p>
                                <span class="text-sm text-gray-500">×{{ item.amount }}</span>
                            </div>
                        </div>
                        <div class="text-right">
                            <p class="font-medium">{{ formatNumber(item.value) }} coins each</p>
                            <p class="text-sm text-gray-600">
                                Total: {{ formatNumber(item.value?.times(item.amount)) }}
                            </p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import type { Item } from '../types'
import BigNumber from 'bignumber.js'

const props = defineProps<{
    show: boolean
    items: Item[]
    packName: string
    formatNumber: (num: BigNumber | number) => string
    packPrice?: number
    showAnimations?: boolean
    hasMorePacks?: boolean
    remainingPacks?: number
}>()

const emit = defineEmits<{
    (e: 'close'): void
    (e: 'openAnother'): void
}>()

const revealedItems = ref<Item[]>([])
const remainingItems = computed(() => props.items.length - revealedItems.value.length)

// Compute stacked items (combine duplicates)
const stackedItems = computed(() => {
    const itemMap = new Map<string, Item>()

    revealedItems.value.forEach(item => {
        const existing = itemMap.get(item.id)
        if (existing) {
            existing.amount += 1
        } else {
            itemMap.set(item.id, { ...item, amount: 1 })
        }
    })

    return Array.from(itemMap.values())
})

const revealNext = () => {
    if (remainingItems.value > 0) {
        const nextItem = props.items[revealedItems.value.length]
        revealedItems.value.push(nextItem)
    }
}

const revealAll = () => {
    if (!props.showAnimations) {
        // Instantly reveal all items
        revealedItems.value = [...props.items]
    } else {
        // Reveal items with animation
        const reveal = () => {
            if (remainingItems.value > 0) {
                revealNext()
                setTimeout(reveal, 200) // Adjust timing as needed
            }
        }
        reveal()
    }
}

// If animations are disabled, reveal all items immediately when modal opens
onMounted(() => {
    if (!props.showAnimations) {
        // Small delay to ensure items are loaded
        setTimeout(() => {
            revealAll()
        }, 50)
    }
})

const close = () => {
    revealedItems.value = []
    emit('close')
}

const openAnother = () => {
    // Clear current items
    revealedItems.value = []
    emit('openAnother')

    // If animations are disabled, automatically reveal all new items after a short delay
    if (!props.showAnimations) {
        // Small delay to ensure new items are loaded
        setTimeout(() => {
            revealAll()
        }, 50)
    }
}

// Calculate total value of revealed items
const totalValue = computed(() => {
    return stackedItems.value.reduce((sum, item) => {
        if (item.value) {
            return sum.plus(item.value.times(item.amount))
        }

        return sum
    }, new BigNumber(0))
})

// Calculate profit/loss
const profit = computed(() => {
    const packCost = props.packPrice ? new BigNumber(props.packPrice) : new BigNumber(0)
    return totalValue.value.minus(packCost)
})

const isProfit = computed(() => {
    return profit.value.isGreaterThanOrEqualTo(0)
})
</script>

<style scoped>
.animate-fade-in {
    animation: v-bind('showAnimations ? "fadeIn 0.5s ease-out" : "none"');
}

@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(10px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

/* Custom scrollbar */
.overflow-y-auto {
    scrollbar-width: thin;
    scrollbar-color: #CBD5E0 #EDF2F7;
}

.overflow-y-auto::-webkit-scrollbar {
    width: 6px;
}

.overflow-y-auto::-webkit-scrollbar-track {
    background: #EDF2F7;
    border-radius: 3px;
}

.overflow-y-auto::-webkit-scrollbar-thumb {
    background-color: #CBD5E0;
    border-radius: 3px;
}
</style>