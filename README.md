# Auto Loan Calculator

An auto loan calculator built with Vue 3, TypeScript, and Tailwind CSS. This application helps users calculate auto loan payments with detailed amortization schedules and visualizations.

## What Are Auto Loans?

Auto loans, also known as car loans, are financial agreements that allow individuals to purchase a vehicle by borrowing money from a lender. The borrower agrees to repay the loan in equal monthly installments over a specified term, with interest. The interest rate is based on the borrower’s credit score, loan term, and other financial factors. Some loans allow for additional payments to reduce the total interest paid or shorten the loan term.

## Current U.S. Auto Loan Market Trends

As of 2024–2025, the U.S. auto loan market is showing signs of strain, especially among [subprime borrowers](https://www.investopedia.com/terms/s/subprime-borrower.asp). Key developments include:

### Rising Auto Loan Debt
- Total U.S. auto loan debt reached **$1.66 trillion** in Q4 2024, up from $1.64 trillion in the previous quarter and is the third largest debt slightly below Student Loans, at **~$1.60 trillion** in the U.S.
- This continued growth reflects both rising vehicle prices and extended loan terms.  

### Rising Delinquency Rates
- The percentage of auto loans **90+ days delinquent** increased to **4.8%** in Q4 2024.
- Though lower than the 5.3% peak seen during the Great Recession (Q4 2010), this is the highest rate in over a decade.

### Subprime Borrower Risk
- **6.6% of subprime auto borrowers** were at least 60 days overdue on their payments as of January 2025.
- This is the highest delinquency rate for this group in decades, highlighting the vulnerability of lower-credit consumers.

### Negative Equity on the Rise
- Nearly **1 in 4** U.S. consumers trading in vehicles in Q3 2024 had [**negative equity**](https://www.chase.com/personal/auto/education/selling/how-to-trade-in-a-car-with-negative-equity).
- At least **20%** of these consumers owed **more than $10,000** over their vehicle’s actual value.
- This trend increases financial risk and may lead to further delinquencies or loan rollovers into new vehicle purchases.

#### References:

[ref: lending tree](https://www.lendingtree.com/auto/debt-statistics/)

[ref: trading economics](https://tradingeconomics.com/united-states/debt-balance-auto-loans)

[ref: bloomberg](https://www.bloomberg.com/news/articles/2025-03-06/late-car-loan-payments-auto-delinquencies-spike-to-highest-level-in-decades)

[ref: investopedia - general](https://www.investopedia.com/terms/n/negativeequity.asp)

## Features

- **Comprehensive Input Fields**:
  - Auto price
  - Interest rate
  - Loan term
  - Down payment
  - Cash incentives
  - Trade-in value
  - Amount owed on trade-in
  - Sales tax
  - Title and registration fees
  - Option to include fees in loan

- **Vehicle Selection**:
  - Integration with NHTSA vPIC API
  - Search for makes, models, and years of vehicles

- **Detailed Calculations**:
  - Monthly payment amount
  - Total loan cost
  - Principal and interest breakdown
  - Complete amortization schedule

- **Visual Representation**:
  - Interactive charts using Chart.js
  - Breakdown of principal, interest, and fees

## Technical Stack

- **Frontend Framework**: Vue 3 with Composition API
- **Language**: TypeScript
- **Build Tool**: Vite
- **CSS Framework**: Tailwind CSS
- **Charting Library**: Chart.js
- **External API**: NHTSA vPIC API for vehicle data

## Setup Instructions

### Prerequisites

- Node.js (v14.x or later)
- npm or pnpm

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/bittricky/auto-loan-calculator.git
   cd auto-loan-calculator
   ```

2. Install dependencies:
   ```bash
   npm install
   # or
   pnpm install
   ```

3. Start the development server:
   ```bash
   npm run dev
   # or
   pnpm run dev
   ```

4. Open your browser and navigate to `http://localhost:5173`

### Building for Production

```bash
npm run build
# or
pnpm run build
```

The built files will be in the `dist` directory.

## Architecture

### Component Structure

The application is built around a single main component (`Calculator.vue`) that handles:

1. **State Management**: Using Vue 3's reactive system
2. **API Integration**: Fetching vehicle data from NHTSA vPIC API
3. **Calculations**: Computing loan amounts, payments, and amortization schedules
4. **UI Rendering**: Displaying inputs, results, and visualizations

### Key Functions

#### Loan Calculation

The application includes functions to calculate monthly payments and generate an amortization schedule based on the [loan amortization formula](https://www.investopedia.com/terms/l/loanamortizationformula.asp):

```typescript
// Calculate monthly payment
const calculateMonthlyPayment = (principal: number, interestRate: number, loanTerm: number) => {
  const monthlyRate = (interestRate / 100) / 12;
  if (monthlyRate === 0) {
    return principal / loanTerm;
  }
  return principal * monthlyRate * Math.pow(1 + monthlyRate, loanTerm) / (Math.pow(1 + monthlyRate, loanTerm) - 1);
}

// Generate amortization schedule
const generateAmortizationSchedule = (principal: number, interestRate: number, loanTerm: number) => {
  const monthlyRate = (interestRate / 100) / 12;
  const monthlyPayment = calculateMonthlyPayment(principal, interestRate, loanTerm);
  let schedule = [];
  let balance = principal;
  
  for (let month = 1; month <= loanTerm; month++) {
    const interestPayment = balance * monthlyRate;
    const principalPayment = monthlyPayment - interestPayment;
    balance -= principalPayment;
    
    schedule.push({
      month,
      payment: monthlyPayment,
      principalPayment,
      interestPayment,
      remainingBalance: balance > 0 ? balance : 0
    });
  }
  
  return schedule;
}
```

#### API Integration

The application uses the [NHTSA vPIC API](https://vpic.nhtsa.dot.gov/api/) to fetch vehicle makes, models, and years:

```typescript
const fetchCarMakes = async () => {
  isLoading.makes = true;
  try {
    const response = await fetch('https://vpic.nhtsa.dot.gov/api/vehicles/getallmakes?format=json');
    const data = await response.json();
    carMakes.value = data.Results;
  } catch (error) {
    console.error('Error fetching car makes:', error);
  } finally {
    isLoading.makes = false;
  }
}
```

### Reactive Data Flow

The application uses Vue 3's `computed` properties to automatically recalculate values when inputs change:

```typescript
const loanAmount = computed(() => {
  const baseAmount = autoPrice.value - downPayment.value - tradeInValue.value + amountOwedOnTradeIn.value - cashIncentives.value;
  const taxAmount = (autoPrice.value - cashIncentives.value) * (salesTax.value / 100);
  
  if (includeFeesInLoan.value) {
    return baseAmount + taxAmount + titleAndRegistrationFees.value;
  } else {
    return baseAmount + taxAmount;
  }
});

const monthlyPayment = computed(() => {
  return calculateMonthlyPayment(loanAmount.value, interestRate.value, loanTerm.value);
});
```

## Usage Guide

1. **Enter Vehicle Information**:
   - Use the car selection dropdown to choose make, model, and year
   - Enter the auto price

2. **Configure Loan Details**:
   - Set loan term and interest rate
   - Add down payment amount
   - Include any cash incentives
   - Enter trade-in value and amount owed on trade-in

3. **Add Tax and Fees**:
   - Select your state
   - Enter sales tax percentage
   - Add title and registration fees
   - Choose whether to include fees in the loan

4. **View Results**:
   - See the calculated monthly payment
   - View the breakdown of principal, interest, and fees
   - Examine the visual chart representation

5. **Check Amortization Schedule**:
   - Click "View Amortization Schedule" to see the payment breakdown by month
   - Review how each payment is applied to principal and interest

## License

MIT

## Author

Mitul Patel
