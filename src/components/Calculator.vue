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

const autoPrice = ref(50000)
const interestRate = ref(5)
const loanTerm = ref(60)
const cashIncentives = ref(0)
const downPayment = ref(10000)
const tradeInValue = ref(0)
const amountOwedOnTradeIn = ref(0)
const salesTax = ref(6)
const titleAndRegistrationFees = ref(4500)
const includeFeesInLoan = ref(false)
const selectedState = ref('Georgia')
const showAmortizationSchedule = ref(false)

const availableTerms = [12, 24, 36, 48, 60, 72, 84]
const availableStates = [
  'Alabama', 'Alaska', 'Arizona', 'Arkansas', 'California', 'Colorado', 'Connecticut', 'Delaware', 'Florida',
  'Georgia', 'Hawaii', 'Idaho', 'Illinois', 'Indiana', 'Iowa', 'Kansas', 'Kentucky', 'Louisiana', 'Maine',
  'Maryland', 'Massachusetts', 'Michigan', 'Minnesota', 'Mississippi', 'Missouri', 'Montana', 'Nebraska',
  'Nevada', 'New Hampshire', 'New Jersey', 'New Mexico', 'New York', 'North Carolina', 'North Dakota', 'Ohio',
  'Oklahoma', 'Oregon', 'Pennsylvania', 'Rhode Island', 'South Carolina', 'South Dakota', 'Tennessee', 'Texas',
  'Utah', 'Vermont', 'Virginia', 'Washington', 'West Virginia', 'Wisconsin', 'Wyoming'
]

const loanAmount = computed(() => {
  const baseAmount = autoPrice.value - downPayment.value - tradeInValue.value + amountOwedOnTradeIn.value - cashIncentives.value
  const taxAmount = (autoPrice.value - cashIncentives.value) * (salesTax.value / 100)
  return includeFeesInLoan.value ? baseAmount + taxAmount + titleAndRegistrationFees.value : baseAmount + taxAmount
})

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
  return loanAmount.value + totalInterest.value + (includeFeesInLoan.value ? 0 : titleAndRegistrationFees.value)
})

const amortizationSchedule = computed(() => {
  return generateAmortizationSchedule(loanAmount.value, interestRate.value, loanTerm.value)
})

onMounted(async () => {
  try {
    await fetchCarMakes()
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

watch([loanAmount, interestRate, loanTerm, titleAndRegistrationFees], () => {
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
                data: [Number(loanAmount.value), totalInterest.value, titleAndRegistrationFees.value],
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

const exportToCSV = () => {
  const headers = ['Month', 'Payment', 'Principal', 'Interest', 'Remaining Balance'];
  const csvContent = [
    headers.join(','),
    ...amortizationSchedule.value.map(payment => [
      payment.month,
      payment.payment.toFixed(2),
      payment.principalPayment.toFixed(2),
      payment.interestPayment.toFixed(2),
      payment.remainingBalance.toFixed(2)
    ].join(','))
  ].join('\n');

  const summary = [
    '\n\nLoan Summary',
    `Vehicle,${selectedMake.value} ${selectedModel.value} ${selectedYear.value}`,
    `Auto Price,$${autoPrice.value.toFixed(2)}`,
    `Down Payment,$${downPayment.value.toFixed(2)}`,
    `Trade-in Value,$${tradeInValue.value.toFixed(2)}`,
    `Amount Owed on Trade-in,$${amountOwedOnTradeIn.value.toFixed(2)}`,
    `Cash Incentives,$${cashIncentives.value.toFixed(2)}`,
    `Sales Tax,${salesTax.value}%`,
    `Title & Registration Fees,$${titleAndRegistrationFees.value.toFixed(2)}`,
    `Loan Amount,$${loanAmount.value.toFixed(2)}`,
    `Interest Rate,${interestRate.value}%`,
    `Loan Term,${loanTerm.value} months`,
    `Monthly Payment,$${monthlyPayment.value.toFixed(2)}`,
    `Total Interest,$${totalInterest.value.toFixed(2)}`,
    `Total Loan Cost,$${totalLoanCost.value.toFixed(2)}`
  ].join('\n');

  const fullCSVContent = csvContent + summary;

  const blob = new Blob([fullCSVContent], { type: 'text/csv;charset=utf-8;' });
  
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  
  const fileName = `${selectedMake.value}_${selectedModel.value}_${selectedYear.value}_loan_calculation.csv`;
  link.setAttribute('href', url);
  link.setAttribute('download', fileName);
  
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
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
        
        <!-- Car selection section (collapsed by default) -->
        <div class="mt-10 border-t pb-6 pt-6">
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
        
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
          <!-- Left side - Inputs -->
          <div class="space-y-6">
            <div>
              <h2 class="text-xl font-medium mb-4">Total Price</h2>
              
              <div class="space-y-4">
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Auto Price</label>
                  <div class="relative">
                    <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
                    <input
                      type="number"
                      v-model="autoPrice"
                      class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    />
                  </div>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Loan Term</label>
                  <div class="relative">
                    <select
                      v-model="loanTerm"
                      class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    >
                      <option v-for="month in availableTerms" :key="month" :value="month">{{ month }} months</option>
                    </select>
                  </div>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Interest Rate</label>
                  <div class="relative">
                    <input
                      type="number"
                      v-model="interestRate"
                      step="0.1"
                      class="w-full pr-8 px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    />
                    <span class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-500">%</span>
                  </div>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Cash Incentives</label>
                  <div class="relative">
                    <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
                    <input
                      type="number"
                      v-model="cashIncentives"
                      class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    />
                  </div>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Down Payment</label>
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
                  <label class="block text-sm text-gray-600 mb-2">Trade-in Value</label>
                  <div class="relative">
                    <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
                    <input
                      type="number"
                      v-model="tradeInValue"
                      class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    />
                  </div>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Amount Owed on Trade-in</label>
                  <div class="relative">
                    <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
                    <input
                      type="number"
                      v-model="amountOwedOnTradeIn"
                      class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    />
                  </div>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Your State</label>
                  <select
                    v-model="selectedState"
                    class="w-full px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                  >
                    <option v-for="state in availableStates" :key="state" :value="state">{{ state }}</option>
                  </select>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Sales Tax</label>
                  <div class="relative">
                    <input
                      type="number"
                      v-model="salesTax"
                      step="0.1"
                      class="w-full pr-8 px-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    />
                    <span class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-500">%</span>
                  </div>
                </div>
                
                <div>
                  <label class="block text-sm text-gray-600 mb-2">Title, Registration and Other Fees</label>
                  <div class="relative">
                    <span class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-500">$</span>
                    <input
                      type="number"
                      v-model="titleAndRegistrationFees"
                      class="w-full pl-8 pr-4 py-2.5 rounded-lg bg-white border border-gray-200 focus:border-red-600 focus:ring-1 focus:ring-red-600"
                    />
                  </div>
                </div>
                
                <div class="flex items-center">
                  <input
                    type="checkbox"
                    id="includeFeesInLoan"
                    v-model="includeFeesInLoan"
                    class="w-5 h-5 text-red-600 rounded focus:ring-red-600"
                  />
                  <label for="includeFeesInLoan" class="ml-2 text-gray-700">Include All Fees in Loan</label>
                </div>
                
                <button
                  @click="createPaymentChart"
                  class="w-full py-3 bg-green-600 text-white font-medium rounded-xl hover:bg-green-700 transition-colors mt-4"
                >
                  Calculate
                </button>
              </div>
            </div>
          </div>
          
          <!-- Right side - Chart and Monthly Payment -->
          <div class="flex flex-col">
            <div class="bg-red-600 text-white p-6 rounded-lg mb-6">
              <h2 class="text-xl font-medium mb-2">Monthly Payment</h2>
              <p class="text-4xl font-bold">{{ formatCurrency(monthlyPayment) }}</p>
            </div>
            
            <div class="w-full h-80 relative bg-white rounded-lg p-4 shadow-sm mb-6">
              <canvas id="paymentChart"></canvas>
              <div class="absolute inset-0 flex items-center justify-center flex-col">
                <p class="text-lg font-medium">Total Cost</p>
                <p class="text-3xl font-bold">{{ formatCurrency(totalLoanCost) }}</p>
              </div>
            </div>
            
            <div class="grid grid-cols-3 gap-4 w-full">
              <div class="flex flex-col items-center bg-white p-3 rounded-lg shadow-sm">
                <div class="flex items-center">
                  <div class="w-3 h-3 rounded-full bg-red-600 mr-2"></div>
                  <span>Principal</span>
                </div>
                <p class="font-bold mt-1">{{ formatCurrency(loanAmount) }}</p>
              </div>
              
              <div class="flex flex-col items-center bg-white p-3 rounded-lg shadow-sm">
                <div class="flex items-center">
                  <div class="w-3 h-3 rounded-full bg-yellow-400 mr-2"></div>
                  <span>Interest</span>
                </div>
                <p class="font-bold mt-1">{{ formatCurrency(totalInterest) }}</p>
              </div>
              
              <div class="flex flex-col items-center bg-white p-3 rounded-lg shadow-sm">
                <div class="flex items-center">
                  <div class="w-3 h-3 rounded-full bg-pink-500 mr-2"></div>
                  <span>Fees</span>
                </div>
                <p class="font-bold mt-1">{{ formatCurrency(titleAndRegistrationFees) }}</p>
              </div>
            </div>
            
            <button
              @click="showAmortizationSchedule = !showAmortizationSchedule"
              class="mt-6 py-2 px-4 bg-gray-100 text-gray-800 font-medium rounded-lg hover:bg-gray-200 transition-colors flex items-center justify-center"
            >
              {{ showAmortizationSchedule ? 'Hide' : 'View' }} Amortization Schedule
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 ml-2" :class="{ 'rotate-180': showAmortizationSchedule }" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
              </svg>
            </button>
            <br />
            <button 
              @click="exportToCSV" 
              class="w-full bg-red-500 hover:bg-red-900 transition-colors text-white font-bold py-3 px-4 rounded-lg flex items-center justify-center mb-6"
            >
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
              </svg>
              Export to CSV
            </button>
          </div>
        </div>
        
        <!-- Amortization Schedule -->
        <div v-if="showAmortizationSchedule" class="mt-8 border-t border-gray-200 pt-6">
          <h2 class="text-xl font-medium mb-4">Amortization Schedule</h2>
          <div class="overflow-x-auto">
            <table class="min-w-full divide-y divide-gray-200">
              <thead>
                <tr>
                  <th class="px-6 py-3 bg-gray-50 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Month</th>
                  <th class="px-6 py-3 bg-gray-50 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Payment</th>
                  <th class="px-6 py-3 bg-gray-50 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Principal</th>
                  <th class="px-6 py-3 bg-gray-50 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Interest</th>
                  <th class="px-6 py-3 bg-gray-50 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Balance</th>
                </tr>
              </thead>
              <tbody class="bg-white divide-y divide-gray-200">
                <tr v-for="(row, index) in amortizationSchedule" :key="index" :class="index % 2 === 0 ? 'bg-white' : 'bg-gray-50'">
                  <td class="px-6 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{{ row.month }}</td>
                  <td class="px-6 py-2 whitespace-nowrap text-sm text-gray-500">{{ formatCurrency(row.payment) }}</td>
                  <td class="px-6 py-2 whitespace-nowrap text-sm text-gray-500">{{ formatCurrency(row.principalPayment) }}</td>
                  <td class="px-6 py-2 whitespace-nowrap text-sm text-gray-500">{{ formatCurrency(row.interestPayment) }}</td>
                  <td class="px-6 py-2 whitespace-nowrap text-sm text-gray-500">{{ formatCurrency(row.remainingBalance) }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
