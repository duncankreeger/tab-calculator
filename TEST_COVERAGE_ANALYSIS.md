# Test Coverage Analysis Report

## Executive Summary

The TAB Loan Calculator codebase currently has **zero test coverage**. There are no test files, no testing frameworks configured, and no CI/CD testing pipeline. Given the financial nature of this application (calculating loan quotes, interest rates, and fees), this represents significant technical risk.

---

## Current State

| Metric | Status |
|--------|--------|
| Test files | 0 |
| Test frameworks | None installed |
| Code coverage | 0% |
| CI/CD testing | None configured |

---

## Critical Areas Requiring Tests

### 1. `parseEnquiry()` - Natural Language Parser (Lines 242-611)
**Priority: CRITICAL**

This function parses freeform text to extract loan enquiry details. It handles:
- Location detection (London postcodes, UK cities/counties, rejected regions)
- Property type detection (HMO, MUFB, commercial types)
- Monetary amount parsing (valuation, rent, loan amounts with k/m suffixes)
- LTV percentage extraction
- Term detection (months/years)
- Product type inference (bridge vs mortgage)
- Borrower type detection

**Risks if untested:**
- Incorrect field extraction leads to wrong quotes
- Edge cases in regex patterns cause silent failures
- Amount parsing errors (e.g., "1.5m" vs "1.5M" vs "1500k")

**Suggested test cases:**
```javascript
// Location detection
parseEnquiry("BTL in SW1A 1AA") // → location: 'london'
parseEnquiry("Property in Manchester") // → location: 'england'
parseEnquiry("House in Belfast") // → location: 'ni', warnings: ['OUT OF POLICY']

// Amount parsing
parseEnquiry("Property worth 500k") // → valuation: 500000
parseEnquiry("£1.5m flat") // → valuation: 1500000
parseEnquiry("Rent £2,500 pcm") // → rent: 2500
parseEnquiry("Rental income £30k pa") // → rent: 2500 (annual converted)

// Property type detection
parseEnquiry("6 bed HMO") // → propertyType: 'hmo'
parseEnquiry("warehouse in Leeds") // → propertyType: 'warehouse', category: 'commercial'

// Product inference
parseEnquiry("Bridge loan needed urgently") // → loanPurpose: 'bridge'
parseEnquiry("BTL mortgage refinance") // → loanPurpose: 'mortgage'
```

---

### 2. `findMaxLoanForRent()` - ICR Calculation (Lines 905-948)
**Priority: CRITICAL**

This function calculates the maximum loan amount that can be supported by a given rental income, subject to ICR (Interest Coverage Ratio) constraints.

**Key logic:**
- Iterates from max LTV down in £1,000 steps
- Different stress rates for Standard vs Pro products
- ICR requirement of 1.25x

**Risks if untested:**
- Calculation errors could result in non-compliant quotes
- Stress rate calculations differ between products
- Edge cases at LTV tier boundaries

**Suggested test cases:**
```javascript
// Basic ICR calculation
findMaxLoanForRent(2500, false) // Standard product
findMaxLoanForRent(2500, true)  // Pro product (should allow higher loan)

// Edge cases
findMaxLoanForRent(0, false)    // Zero rent → loan: 0
findMaxLoanForRent(100, false)  // Very low rent → limited by ICR

// LTV tier boundaries
// Test at 60%, 65%, 70% LTV boundaries
```

---

### 3. `buildQuote()` - Quote Generation (Lines 1005-1060)
**Priority: HIGH**

This function generates complete loan quotes including:
- LTV calculation
- Rate margin lookup
- Fee calculations (arrangement, admin, valuation, exit)
- ESG discount application
- Interest calculations (standard vs prepaid)
- Net advance calculations

**Risks if untested:**
- Incorrect fee calculations affect business revenue
- ESG discount logic errors
- Interest calculation differences between serviced/retained

**Suggested test cases:**
```javascript
// Fee calculations
buildQuote(500000, false, false) // Check arrangement fee = 2%
buildQuote(500000, true, false)  // Pro should have higher admin fee

// ESG discounts
// With EPC + sustainability + social discounts applied

// Interest calculations
// Standard: monthly on pay rate
// Pro: monthly on effective rate (after prepaid deducted)
```

---

### 4. `getMargin()` - Rate Table Lookup (Lines 861-866)
**Priority: HIGH**

Simple but critical function that looks up the margin for a given LTV in rate tables.

**Suggested test cases:**
```javascript
// Boundary testing
getMargin(60, rates)  // At tier boundary
getMargin(60.1, rates) // Just above tier boundary
getMargin(75, rates)  // Above max tier → null
```

---

### 5. `getBridgeRates()` - Rate Table Selection (Lines 846-851)
**Priority: MEDIUM**

Selects the appropriate bridge rate table based on loan amount.

**Suggested test cases:**
```javascript
getBridgeRates(499999) // Under £500k rates
getBridgeRates(500000) // Over £500k rates
getBridgeRates(500001) // Over £500k rates
```

---

### 6. `buildCorePlusQuote()` - Core Plus Product (Lines 1078-1121)
**Priority: HIGH**

Generates quotes for Core Plus products with:
- Rate loading for various conditions
- 1st vs 2nd charge variants
- PCM rate conversion

**Suggested test cases:**
```javascript
// Rate loading
buildCorePlusQuote({ corePlusLoading: { lightRefurb: true } }) // +0.10%
buildCorePlusQuote({ corePlusLoading: { heavyRefurb: true, adverse: true } }) // +0.30%

// Charge type
buildCorePlusQuote({ corePlusCharge: 'first' })
buildCorePlusQuote({ corePlusCharge: 'second' })
```

---

### 7. `findRentForMaxLtv()` - Rent Calculation (Lines 870-888)
**Priority: HIGH**

Calculates the minimum rent required to support maximum LTV.

**Suggested test cases:**
```javascript
findRentForMaxLtv(false) // Standard product
findRentForMaxLtv(true)  // Pro product (should require less rent)
```

---

### 8. Formatting Functions (`fmt`) (Lines 613-617)
**Priority: LOW**

Currency and percentage formatting utilities.

**Suggested test cases:**
```javascript
fmt.currency(500000)     // "£500,000"
fmt.currency(1234567)    // "£1,234,567"
fmt.currencyFull(500.50) // "£500.50"
fmt.percent(5.25)        // "5.25%"
```

---

## React Component Testing Needs

### `App` Component (Lines 660-2670)
**Priority: MEDIUM**

While the business logic tests are more critical, component tests would help ensure:
- Form inputs properly update state
- Section collapse/expand behavior
- Quote display renders correctly
- Admin panel configuration saves properly

---

## Recommended Testing Setup

### 1. Install Testing Dependencies

```bash
npm init -y
npm install --save-dev jest @testing-library/react @testing-library/jest-dom
```

### 2. Refactor for Testability

The current single-file architecture makes testing difficult. Recommend extracting:

```
src/
├── utils/
│   ├── parseEnquiry.js
│   ├── calculations.js
│   └── formatters.js
├── components/
│   ├── Section.jsx
│   ├── OptionButton.jsx
│   └── InputField.jsx
├── App.jsx
└── index.js
```

### 3. Add Test Files

```
src/
├── utils/
│   ├── parseEnquiry.js
│   ├── parseEnquiry.test.js
│   ├── calculations.js
│   ├── calculations.test.js
│   ├── formatters.js
│   └── formatters.test.js
```

### 4. Jest Configuration

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['@testing-library/jest-dom'],
  moduleFileExtensions: ['js', 'jsx'],
  testMatch: ['**/*.test.js', '**/*.test.jsx'],
  collectCoverageFrom: ['src/**/*.{js,jsx}'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

### 5. NPM Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

---

## Coverage Targets

| Category | Target | Rationale |
|----------|--------|-----------|
| Financial calculations | 100% | Business-critical, compliance risk |
| Input parsing | 95% | User-facing, many edge cases |
| Rate lookups | 100% | Business-critical |
| Formatting | 80% | Lower risk, simple logic |
| UI components | 60% | Important but lower priority |

---

## Implementation Priority

### Phase 1 (Immediate)
1. Set up Jest testing framework
2. Extract `parseEnquiry()` and add tests
3. Extract calculation functions and add tests
4. Add tests for rate lookup functions

### Phase 2 (Short-term)
1. Extract and test formatting utilities
2. Add integration tests for quote generation flow
3. Set up code coverage reporting

### Phase 3 (Medium-term)
1. Add React component tests
2. Set up CI/CD with test requirements
3. Add E2E tests with Cypress or Playwright

---

## Risk Assessment

| Area | Current Risk | After Testing |
|------|-------------|---------------|
| Loan calculations | HIGH | LOW |
| ICR compliance | HIGH | LOW |
| Fee calculations | MEDIUM | LOW |
| Input parsing | MEDIUM | LOW |
| UI rendering | LOW | LOW |

---

## Conclusion

The lack of test coverage in this financial application represents significant business and compliance risk. The `parseEnquiry()`, `findMaxLoanForRent()`, and `buildQuote()` functions should be prioritized for testing as they directly affect loan quotes and customer-facing calculations.

Implementing a testing framework and achieving 80%+ coverage on critical calculation functions would significantly reduce the risk of bugs affecting loan quotes and business operations.
