# Form Validation — Block Save on Invalid Data

Groovy script that validates user input on an Oracle EPM Cloud Planning web form and blocks the save operation if business rules are violated, displaying a custom error message to the user.

Developed for a production Planning environment where users were submitting forecast values without mandatory fields populated, causing downstream calculation errors.

---

## Problem

Native EPM form validations are limited to required fields and basic data type checks. More complex business validations — such as preventing a save when a value exceeds a threshold, or when a supporting account is empty — require Groovy scripts.

Without this validation, invalid data entered on the form propagates silently to the Essbase cube and corrupts downstream calculations.

---

## Script

```groovy
// Form Validation — Block Save on Invalid Data
// Triggered: On Save (before data is written to the cube)

def grid = operation.getGrid()
def hasError = false
def errorMessages = []

grid.dataCellIterator("Account").each { cell ->

    def account = cell.getMemberName("Account")
    def value = cell.getValue()

    // Validation 1: Block save if value is negative for revenue accounts
    if (account.startsWith("REV_") && value != null && value < 0) {
        hasError = true
        errorMessages.add("Account ${account}: Revenue values cannot be negative. Found: ${value}")
    }

    // Validation 2: Block save if value exceeds budget tolerance (110% of prior year)
    def priorYear = cell.getCrossRef("FY2024", account) as Double
    if (priorYear != null && priorYear > 0 && value != null) {
        def tolerance = priorYear * 1.10
        if (value > tolerance) {
            hasError = true
            errorMessages.add("Account ${account}: Value ${value} exceeds 110% of prior year (${priorYear}). Review before saving.")
        }
    }
}

// Block save and display errors if any validation failed
if (hasError) {
    def fullMessage = errorMessages.join("\n")
    operation.logMessage("Validation failed:\n${fullMessage}")
    throw new Exception("Save blocked — validation errors:\n${fullMessage}")
}

operation.logMessage("Form validation passed. Proceeding with save.")
```

---

## Validation Rules in This Script

| Rule | Condition | Action |
|---|---|---|
| Negative revenue | Account starts with `REV_` and value < 0 | Block save + error message |
| Budget tolerance | Value > 110% of prior year | Block save + error message |

---

## How to Customize

- Replace `REV_` with your actual account prefix or member name pattern
- Replace `FY2024` with the prior year dimension member in your application
- Add or remove validation blocks following the same pattern
- Adjust the tolerance percentage (currently 1.10 = 110%) to match your business rules

---

## How to Attach to a Form

1. Open Calculation Manager
2. Create a new Groovy rule
3. Paste this script
4. In the Planning form designer, attach the rule to the **On Save** event
5. Test in DEV with valid and invalid data to confirm blocking behavior

---

## Notes

- The `throw new Exception()` call is what actually blocks the save — without it, the script logs the error but allows the save to proceed
- Error messages appear in the Job Console and, depending on EPM version, may surface directly to the user on the form
- Always test with multiple users simultaneously to validate there are no concurrency issues

---

## Environment

- Oracle EPM Cloud Planning (PBCS / EPBCS)
- Groovy Engine 26.05+
- Calculation Manager — Form Event (On Save)
