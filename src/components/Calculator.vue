<script setup lang="ts">

import { ref, computed, onMounted, watch } from 'vue'
import { Chart, registerables } from 'chart.js';

Chart.register(...registerables);

interface CarMake {
  MakeId: number
  MakeName: string
}

interface CarModel {
  Model_ID: number
  Model_Name: string
}

const carMakes = ref<CarMake[]>([])
const availableModels = ref<CarModel[]>([])
const availableYears = ref<number[]>([])
const isLoading = ref({
  makes: true,
  models: false,
  years: false
})

const selectedMake = ref('')
const selectedMakeId = ref(0)
const selectedModel = ref('')
const selectedModelId = ref(0)
const selectedYear = ref(new Date().getFullYear())
const showCarSelection = ref(false)

// Loan calculator variables
const loanAmount = ref(25000)
const interestRate = ref(5.9)
const loanTerm = ref(24)
const fees = ref(2500)
const availableTerms = [12, 24, 36, 48, 60, 72, 84]

// Computed values for loan calculation
const monthlyPayment = computed(() => calculateMonthlyPayment(
    loanAmount.value,
    interestRate.value,
    loanTerm.value
))

const totalInterest = computed(() => calculateTotalInterest(
    loanAmount.value,
    monthlyPayment.value,
    loanTerm.value
))

const totalLoanCost = computed(() => {
  return loanAmount.value + totalInterest.value + fees.value
})

onMounted(async () => {
  try {
    await fetchCarMakes()
    // Initialize the payment chart
    createPaymentChart()
  } catch (error) {
    console.error('Failed to fetch car makes:', error)
  }
})

async function fetchCarMakes() {
  isLoading.value.makes = true
  try {
    const response = await fetch('https://vpic.nhtsa.dot.gov/api/vehicles/GetMakesForVehicleType/car?format=json')
    const data = await response.json()
    
    carMakes.value = data.Results.sort((a: CarMake, b: CarMake) => {
      return a.MakeName.localeCompare(b.MakeName)
    })
    
    if (carMakes.value.length > 0) {
      selectedMake.value = carMakes.value[0].MakeName
      selectedMakeId.value = carMakes.value[0].MakeId
      await fetchModels(selectedMakeId.value)
    }
  } catch (error) {
    console.error('Error fetching car makes:', error)
  } finally {
    isLoading.value.makes = false
  }
}

async function fetchModels(makeId: number) {
  isLoading.value.models = true
  availableModels.value = []
  try {
    const response = await fetch(`https://vpic.nhtsa.dot.gov/api/vehicles/GetModelsForMakeId/${makeId}?format=json`)
    const data = await response.json()
    
    availableModels.value = data.Results.sort((a: CarModel, b: CarModel) => {
      return a.Model_Name.localeCompare(b.Model_Name)
    })
    
    if (availableModels.value.length > 0) {
      selectedModel.value = availableModels.value[0].Model_Name
      selectedModelId.value = availableModels.value[0].Model_ID
      await fetchYears(selectedMakeId.value, selectedModel.value)
    }
  } catch (error) {
    console.error('Error fetching models:', error)
  } finally {
    isLoading.value.models = false
  }
}

async function fetchYears(makeId: number, modelName: string) {
  isLoading.value.years = true
  availableYears.value = []
  try {
    const currentYear = new Date().getFullYear()
    const years = []
    
    for (let year = currentYear; year >= currentYear - 10; year--) {
      try {
        const response = await fetch(`https://vpic.nhtsa.dot.gov/api/vehicles/GetModelsForMakeIdYear/makeId/${makeId}/modelyear/${year}?format=json`)
        const data = await response.json()
        
        const modelExists = data.Results.some((m: any) => m.Model_Name.toLowerCase() === modelName.toLowerCase())
        
        if (modelExists) {
          years.push(year)
        }
      } catch (error) {
        console.error(`Error checking year ${year}:`, error)
      }
    }
    
    if (years.length === 0) {
      for (let year = currentYear; year >= currentYear - 5; year--) {
        years.push(year)
      }
    }
    
    availableYears.value = years.sort((a, b) => b - a)
    
    if (availableYears.value.length > 0) {
      selectedYear.value = availableYears.value[0]
    }
  } catch (error) {
    console.error('Error fetching years:', error)
  } finally {
    isLoading.value.years = false
  }
}

watch(selectedMake, async (newMake) => {
  const make = carMakes.value.find(m => m.MakeName === newMake)
  if (make) {
    selectedMakeId.value = make.MakeId
    await fetchModels(make.MakeId)
  }
})

watch(selectedModel, async (newModel) => {
  const model = availableModels.value.find(m => m.Model_Name === newModel)
  if (model) {
    selectedModelId.value = model.Model_ID
    await fetchYears(selectedMakeId.value, newModel)
  }
})

// Watch for changes in loan calculation values to update the chart
watch([loanAmount, interestRate, loanTerm, fees], () => {
  createPaymentChart()
})

const calculateMonthlyPayment = (principal: number, interestRate: number, loanTerm: number) => {
  const monthlyRate = (interestRate / 100) / 12

  if (monthlyRate === 0) {
    return principal / loanTerm
  }

  // The Math for Auto Loans: P * r * (1 + r)^n / ((1 + r)^n - 1)
  return principal * monthlyRate * Math.pow(1 + monthlyRate, loanTerm) / (Math.pow(1 + monthlyRate, loanTerm) - 1)
}

const generateAmortizationSchedule = (principle: number, interestRate: number, loanTerm: number) => {
    const monthlyRate = (interestRate / 100) / 12
    const monthlyPayment = calculateMonthlyPayment(principle, interestRate, loanTerm)

    let schedule = []
    let balance = principle

    for (let month = 1; month <= loanTerm; month++) {
        const interestPayment = balance * monthlyRate
        const principalPayment = monthlyPayment - interestPayment

        balance -= principalPayment;

        schedule.push({
            month,
            payment: monthlyPayment, 
            principalPayment,
            interestPayment,
            remainingBalance: balance > 0 ? balance : 0
        })
    }
  
    return schedule
}

const createPaymentChart = () => {
    const ctx = document.getElementById('paymentChart')?.getContext('2d')
    if (!ctx) return
    
    const chartStatus = Chart.getChart('paymentChart')
    if (chartStatus) chartStatus.destroy()
    
    new Chart(ctx, {
        type: 'doughnut',
        data: {
            labels: ['Principal', 'Interest', 'Fees'],
            datasets: [{
                data: [Number(loanAmount.value), totalInterest.value, fees.value],
                backgroundColor: ['#dc2626', '#facc15', '#ec4899'],
                borderWidth: 0
            }]
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            cutout: '70%',
            plugins: {
                legend: {
                    display: false
                },
                tooltip: {
                    callbacks: {
                        label: function(context: any) {
                            const label = context.label || '';
                            const value = context.raw ? Number(context.raw) : 0;
                            return `${label}: ${formatCurrency(value)}`;
                        }
                    }
                }
            }
        }
    })
}

const calculateTotalInterest = (principal: number, monthlyPayment: number, loanTerm: number) => {
  return (monthlyPayment * loanTerm) - principal
}

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
    <div class="max-w-5xl mx-auto">
      <div class="bg-white/95 rounded-3xl shadow-xl p-8">
        <h1 class="text-3xl font-bold text-center mb-4">Auto Loan Calculator</h1>
        <p class="text-center text-gray-600 mb-10">The calculated amount is approximate. The exact terms of the loan are determined individually.</p>
        
        <div class="border-t border-gray-200 my-6"></div>
        
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
          <!-- Left side - Inputs -->
          <div class="space-y-8">
            <div>
              <h2 class="text-xl font-medium mb-4">How much money do you need?</h2>
              <div class="relative">
                <input
                  type="text"
                  v-model="loanAmount"
                  class="w-full px-4 py-3 rounded-lg bg-gray-50 border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                />
                <span class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
              </div>
              <div class="mt-4">
                <input
                  type="range"
                  v-model="loanAmount"
                  min="1000"
                  max="50000"
                  step="500"
                  class="w-full h-2 bg-red-600 rounded-lg appearance-none cursor-pointer"
                />
              </div>
            </div>
            
            <div>
              <h2 class="text-xl font-medium mb-4">How many months do you need?</h2>
              <select
                v-model="loanTerm"
                class="w-full px-4 py-3 rounded-lg bg-gray-50 border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
              >
                <option v-for="month in availableTerms" :key="month" :value="month">{{ month }}</option>
              </select>
            </div>
            
            <div>
              <h2 class="text-xl font-medium mb-4">Interest Rate (%)</h2>
              <div class="relative">
                <input
                  type="range"
                  v-model="interestRate"
                  min="0"
                  max="20"
                  step="0.25"
                  class="w-full h-2 bg-red-600 rounded-lg appearance-none cursor-pointer"
                />
                <div class="mt-2 text-lg font-medium">{{ interestRate }}%</div>
              </div>
            </div>
            
            <div class="bg-gray-50 rounded-xl p-6">
              <h3 class="text-center text-lg font-medium mb-2">You Will Pay</h3>
              <p class="text-center text-3xl font-bold text-red-600 mb-1">{{ formatCurrency(monthlyPayment) }}</p>
              <p class="text-center text-gray-500">Per Month</p>
            </div>
          </div>
          
          <!-- Right side - Chart -->
          <div class="flex flex-col items-center justify-center">
            <div class="w-full h-80 relative">
              <canvas id="paymentChart"></canvas>
              <div class="absolute inset-0 flex items-center justify-center flex-col">
                <p class="text-lg font-medium">In Total</p>
                <p class="text-3xl font-bold">{{ formatCurrency(totalLoanCost) }}</p>
              </div>
            </div>
            
            <div class="grid grid-cols-3 gap-8 w-full mt-8">
              <div class="flex flex-col items-center">
                <div class="flex items-center">
                  <div class="w-3 h-3 rounded-full bg-red-600 mr-2"></div>
                  <span>Principal</span>
                </div>
                <p class="font-bold mt-1">{{ formatCurrency(loanAmount) }}</p>
              </div>
              
              <div class="flex flex-col items-center">
                <div class="flex items-center">
                  <div class="w-3 h-3 rounded-full bg-yellow-400 mr-2"></div>
                  <span>Interest</span>
                </div>
                <p class="font-bold mt-1">{{ formatCurrency(totalInterest) }}</p>
              </div>
              
              <div class="flex flex-col items-center">
                <div class="flex items-center">
                  <div class="w-3 h-3 rounded-full bg-pink-500 mr-2"></div>
                  <span>Fees</span>
                </div>
                <p class="font-bold mt-1">{{ formatCurrency(fees) }}</p>
              </div>
            </div>
          </div>
        </div>
        
        <!-- Car selection section (collapsed by default) -->
        <div class="mt-10 border-t border-gray-200 pt-6">
          <button @click="showCarSelection = !showCarSelection" class="flex items-center text-red-600 font-medium">
            <span>{{ showCarSelection ? 'Hide' : 'Show' }} Car Selection</span>
            <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 ml-2" :class="{ 'rotate-180': showCarSelection }" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
            </svg>
          </button>
          
          <div v-if="showCarSelection" class="mt-4 grid grid-cols-3 gap-4">
            <div>
              <label class="block text-sm text-gray-600 mb-2">Select Make</label>
              <div v-if="isLoading.makes" class="text-sm text-gray-500 italic">Loading makes...</div>
              <select
                v-else
                v-model="selectedMake"
                class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
              >
                <option v-for="make in carMakes" :key="make.MakeId" :value="make.MakeName">
                  {{ make.MakeName }}
                </option>
              </select>
            </div>

            <div>
              <label class="block text-sm text-gray-600 mb-2">Select Model</label>
              <div v-if="isLoading.models" class="text-sm text-gray-500 italic">Loading models...</div>
              <select
                v-else
                v-model="selectedModel"
                class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                :disabled="availableModels.length === 0"
              >
                <option v-for="model in availableModels" :key="model.Model_ID" :value="model.Model_Name">
                  {{ model.Model_Name }}
                </option>
              </select>
            </div>

            <div>
              <label class="block text-sm text-gray-600 mb-2">Select Year</label>
              <div v-if="isLoading.years" class="text-sm text-gray-500 italic">Loading years...</div>
              <select
                v-else
                v-model="selectedYear"
                class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                :disabled="availableYears.length === 0"
              >
                <option v-for="year in availableYears" :key="year" :value="year">{{ year }}</option>
              </select>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
