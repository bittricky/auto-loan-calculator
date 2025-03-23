<script setup lang="ts">

import { ref, computed } from 'vue'
import { Switch } from '@headlessui/vue'

interface CarMake {
  id: number
  name: string
  models: string[]
}

const carMakes: CarMake[] = [
  { id: 1, name: 'Range Rover', models: ['Evoque', 'Sport', 'Velar', 'Defender'] },
  { id: 2, name: 'BMW', models: ['3 Series', '5 Series', 'X3', 'X5'] },
  { id: 3, name: 'Mercedes', models: ['C-Class', 'E-Class', 'GLC', 'GLE'] },
  { id: 4, name: 'Audi', models: ['A4', 'A6', 'Q5', 'Q7'] },
]

const selectedMake = ref('Range Rover')
const selectedModel = ref('Evoque')
const selectedYear = ref(2020)
const vehiclePrice = ref(7000)
const sellingPrice = ref(9000)
const downPayment = ref(2000)
const loanTerm = ref(3)
const isAutoFinance = ref(true)

const years = computed(() => {
  const currentYear = new Date().getFullYear()
  const years = []

  for (let year = 2015; year <= currentYear; year++) {
    years.push(year)
  }
  
  return years
})

const availableModels = computed(() => {
  const make = carMakes.find(m => m.name === selectedMake.value)
  return make ? make.models : []
})

const profitMargin = computed(() => {
  const profit = sellingPrice.value - vehiclePrice.value
  return Math.round((profit / vehiclePrice.value) * 100)
})

const totalProfit = computed(() => {
  return sellingPrice.value - vehiclePrice.value
})

const formatCurrency = (value: number) => {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 0,
    maximumFractionDigits: 0
  }).format(value)
}
</script>

<template>
  <div class="min-h-screen bg-gradient-to-r from-red-600 to-yellow-400 py-12 px-4 sm:px-6 lg:px-8">
    <div class="max-w-4xl mx-auto">
      <h1 class="text-5xl font-bold text-white mb-12 text-center drop-shadow-lg">Car Loan Calculator</h1>
      
      <div class="bg-white rounded-3xl shadow-xl p-8">
        <h2 class="text-xl font-semibold mb-6">Calculate your profit</h2>
        
        <div class="grid grid-cols-3 gap-4 mb-8">
          <div>
            <label class="block text-sm text-gray-600 mb-2">Select Make</label>
            <select
              v-model="selectedMake"
              class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
            >
              <option v-for="make in carMakes" :key="make.id" :value="make.name">
                {{ make.name }}
              </option>
            </select>
          </div>

          <div>
            <label class="block text-sm text-gray-600 mb-2">Select Model</label>
            <select
              v-model="selectedModel"
              class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
            >
              <option v-for="model in availableModels" :key="model" :value="model">
                {{ model }}
              </option>
            </select>
          </div>

          <div>
            <label class="block text-sm text-gray-600 mb-2">Select Year</label>
            <select
              v-model="selectedYear"
              class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
            >
              <option v-for="year in years" :key="year" :value="year">{{ year }}</option>
            </select>
          </div>
        </div>

        <div class="space-y-6">
          <div>
            <label class="block text-sm text-gray-600 mb-2">Total cost of car</label>
            <div class="relative">
              <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
              <input
                type="number"
                v-model="vehiclePrice"
                class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
              />
            </div>
          </div>

          <div>
            <label class="block text-sm text-gray-600 mb-2">Selling price</label>
            <div class="relative">
              <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
              <input
                type="number"
                v-model="sellingPrice"
                class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
              />
            </div>
          </div>

          <div>
            <label class="block text-sm text-gray-600 mb-2">What is your down payment?</label>
            <div class="relative">
              <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
              <input
                type="number"
                v-model="downPayment"
                class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
              />
            </div>
          </div>

          <div>
            <label class="block text-sm text-gray-600 mb-2">Loan Duration</label>
            <div class="relative">
              <input
                type="range"
                v-model="loanTerm"
                min="1"
                max="12"
                class="w-full h-2 bg-red-600 rounded-lg appearance-none cursor-pointer"
              />
              <div class="mt-2 text-sm text-gray-600">{{ loanTerm }} Months</div>
            </div>
          </div>
        </div>

        <div class="flex gap-4 mt-8">
          <div class="flex-1 bg-red-100 rounded-2xl p-6">
            <button class="text-red-600 font-medium mb-4">Auto-Finance</button>
            <div class="text-4xl font-bold text-red-600 mb-4">{{ formatCurrency(sellingPrice) }}</div>
            <div class="bg-red-600 text-white rounded-xl p-4">
              <div class="text-lg font-semibold mb-1">{{ profitMargin }}% profit margin</div>
              <div class="text-sm opacity-90">
                Earn {{ formatCurrency(totalProfit) }} profit on {{ formatCurrency(vehiclePrice) }}
              </div>
            </div>
            <div class="mt-4 space-y-2">
              <div class="flex justify-between text-sm">
                <span class="text-gray-600">Admin fee</span>
                <span class="font-medium">$100</span>
              </div>
              <div class="flex justify-between text-sm">
                <span class="text-gray-600">Insurance</span>
                <span class="font-medium">$45</span>
              </div>
              <div class="flex justify-between text-sm">
                <span class="text-gray-600">Interest rate</span>
                <span class="font-medium">0%</span>
              </div>
            </div>
            <button class="w-full mt-6 py-3 bg-white text-red-600 font-medium rounded-xl hover:bg-red-50 transition-colors">
              Proceed
            </button>
          </div>

          <div class="flex-1 bg-yellow-100 rounded-2xl p-6">
            <button class="text-yellow-600 font-medium mb-4">Full Payment</button>
            <div class="text-4xl font-bold text-yellow-600 mb-4">{{ formatCurrency(sellingPrice) }}</div>
            <div class="bg-yellow-200 text-yellow-700 rounded-xl p-4">
              <div class="text-lg font-semibold mb-1">{{ profitMargin }}% profit margin</div>
              <div class="text-sm opacity-90">
                Earn {{ formatCurrency(totalProfit) }} profit on {{ formatCurrency(vehiclePrice) }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
