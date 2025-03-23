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
