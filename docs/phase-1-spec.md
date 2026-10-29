# Phase 1 spec: ISA maturity instructions

## 1. Problem

When an ISA matures, the member has to tell the building society what to do with the money. In a typical paper-based process, the building society posts the maturity details, the member posts back an instruction, and staff process each instruction manually. That means repeated manual work and handovers where mistakes creep in. It also means an invalid instruction (for example, withdrawing more than the balance) is only discovered when someone processes it.

**Phase 1 goal:** an API that the mobile app can call to submit a maturity instruction, choosing one of four options:

1. reinvest the whole balance into a new ISA product
2. add more money and reinvest
3. withdraw some of the funds and reinvest the rest
4. withdraw everything

The API validates each instruction against ISA rules when it is submitted. Valid instructions are recorded; invalid ones are rejected immediately with a clear reason. This removes the late discovery of invalid instructions. Carrying out valid instructions stays manual until phase 2.

## 2. Scope (in and out)

**In scope (phase 1):**

- a member submits a maturity instruction for one of their ISAs, choosing one of the four options
- a member changes their instruction while the instruction window is open
- the API validates each instruction against the rules in section 3
- valid instructions are recorded; invalid ones are rejected with a clear reason and are not stored
- read endpoints that give the app what it needs to show the member their choices (section 5)

**Out of scope (phase 1):**

- moving money: no "add money" payments and no withdrawal payouts (phase 2)
- carrying out instructions on the maturity date (phase 2)
- the default when no instruction is given: the balance moves into an easy-access ISA so it keeps its tax-free status. This needs a scheduled job (phase 2)
- payment states and idempotency (phase 2)
- storing rejected attempts. Rejections are logged with a reason code instead. If an audit trail of attempts is needed later (for example, to show members weren't blocked unfairly), it gets its own table rather than mixing invalid data into the instructions table
- maturity-exclusive products and rates, which real providers offer only to members with a maturing ISA. They add pricing and eligibility rules without changing the core design
- deposit windows on the new product (once its funding window closes, it is locked until its own maturity date)
- adding a new nominated bank account (needs security checks and Confirmation of Payee name matching)
- authentication: phase 1 identifies the member by an ID in the request path. A real service would take it from a verified login token
- deployment to AWS (phase 3)

## 3. Domain rules

Rules are checked in the order R0 to R5. Each rejection has a reason code, returned as described in section 6.

### R0. The ISA

- The ISA must exist and belong to the member making the request: otherwise `ISA_NOT_FOUND`. A member asking about someone else's ISA gets the same response, so the API doesn't reveal that the ISA exists.
- The ISA must be maturing: it has a maturity date and its status is `OPEN`: otherwise `ISA_NOT_MATURING`.

### R1. Instruction window

An instruction can only be submitted within the instruction window. All dates and times are UK time (Europe/London).

- The window **opens** at the start of the day a configurable number of days before the maturity date (default: 30 days).
- The window **closes** at midnight at the end of the day before the maturity date (default: 1 day before). For an ISA maturing on 15 March, the last moment to submit is 23:59:59 on 14 March.

Outside the window, the instruction is rejected:

- before the window opens: `INSTRUCTION_WINDOW_NOT_OPEN`
- after the window closes: `INSTRUCTION_WINDOW_CLOSED`

The defaults reflect a typical provider (around 14 to 30 days before maturity). They are configuration so they can change without a code change, for example to close nearer the maturity date now that instructions no longer need manual processing.

### R2. Changing an instruction

A member can change their instruction while the window is open by submitting a new one.

- Each maturing ISA has at most **one active instruction**.
- A new valid instruction becomes active, and the previous one is marked **superseded** with the time it was replaced. Instructions are never deleted, so there is a history of what was in force and when.
- An invalid new instruction is rejected and leaves the current active instruction unchanged.
- Once the window closes, the active instruction can no longer be changed (R1).

### R3. Reinvestment products

Options 1 to 3 reinvest money into an ISA product chosen from a product list.

- Products are reference data (for example, a 1-year fixed rate cash ISA, a 2-year fixed rate cash ISA, an easy-access cash ISA, a limited-access cash ISA), loaded by a database migration.
- Each product has a **minimum deposit** (currently £1 for all products, but stored per product) and is either **available** or **withdrawn** from sale.
- The chosen product must exist and be available: otherwise `PRODUCT_NOT_AVAILABLE`.
- The amount being reinvested must meet the product's minimum deposit: otherwise `BELOW_PRODUCT_MINIMUM`. For option 3, this applies to the amount left after the withdrawal.
- Option 4 (withdraw everything) has no product.

### R4. Adding money (option 2)

- The amount must be greater than zero: otherwise `INVALID_AMOUNT`.
- New money needs a valid ISA declaration, so the request must confirm the member accepts it: otherwise `ISA_DECLARATION_REQUIRED`.
- New money counts towards the annual ISA allowance; reinvesting the matured balance does not.
- The amount the member can add is their **available to deposit** amount for the tax year:
    - **remaining allowance** = the annual allowance minus new money already paid into the member's ISAs with this building society in that tax year
    - **replaceable flexible withdrawals** = money withdrawn from a **flexible** ISA earlier in the same tax year that hasn't been paid back yet. Replacing it doesn't use allowance
    - **available to deposit** = remaining allowance + replaceable flexible withdrawals
- Adding more than the available amount is rejected: `EXCEEDS_AVAILABLE_TO_DEPOSIT`.
- The tax year is the one containing the **maturity date**, because that's when the money would be paid in. The allowance comes from configuration for that tax year (R6).
- Payments to other providers can't be seen. The member's ISA declaration covers them.

**Example:** a flexible ISA, with £5,000 of new money paid in this tax year and £10,000 withdrawn earlier this tax year. Available to deposit = (£20,000 − £5,000) + £10,000 = £25,000. Adding £25,000 is accepted; adding £25,000.01 is rejected.

### R5. Withdrawals (options 3 and 4)

- Withdrawn money goes to a destination the member **already has on file**: either another of their accounts with the building society, or their nominated bank account. Otherwise: `INVALID_DESTINATION`.
- **Option 3 (withdraw some):** the amount must be greater than zero: otherwise `INVALID_AMOUNT`. It must be less than the expected maturity balance: otherwise `WITHDRAWAL_EXCEEDS_BALANCE` (withdrawing the full balance is option 4). The remainder must meet the chosen product's minimum deposit (R3).
- **Option 4 (withdraw everything):** no amount is given; the whole maturity balance is withdrawn.
- Validation uses the **expected maturity balance** stored for the ISA, as the final balance (including interest) is only known on the maturity date. Phase 2 must handle any small difference when the instruction is carried out.

### R6. Configuration

These values are configuration, not code, because they vary between providers and change with government policy:

- the instruction window: days before maturity it opens (default 30) and closes (default 1)
- the annual ISA allowance for each tax year (for example, 2026/27: £20,000)

## 4. Data model

| Entity | Fields |
|---|---|
| **Member** | `id`, `name` |
| **ISA** | `id`, `memberId`, `productId` (its current product), `maturityDate`, `expectedMaturityBalance`, `status` (`OPEN`, `MATURED`, `CLOSED`) |
| **Product** | `id`, `name`, `type` (`FIXED_RATE`, `EASY_ACCESS`, `LIMITED_ACCESS`), `termMonths` (fixed rate only), `minimumDeposit`, `flexible`, `available` |
| **Destination account** | `id`, `memberId`, `type` (`SOCIETY_ACCOUNT`, `NOMINATED_BANK_ACCOUNT`), `displayName`, `maskedAccountNumber` (for example `****1234`) |
| **Tax year subscription summary** | `memberId`, `taxYear` (for example `2026/27`), `newMoneyPaidIn`, `flexibleWithdrawalsNotReplaced` |
| **Maturity instruction** | `id`, `isaId`, `option`, `productId`, `addAmount`, `isaDeclarationAccepted`, `withdrawalAmount`, `destinationAccountId`, `status` (`ACTIVE`, `SUPERSEDED`), `submittedAt`, `supersededAt` |

**Which instruction fields each option uses:**

| Option | `productId` | `addAmount` | `isaDeclarationAccepted` | `withdrawalAmount` | `destinationAccountId` |
|---|---|---|---|---|---|
| `REINVEST_ALL` | required | none | none | none | none |
| `ADD_AND_REINVEST` | required | required | required (true) | none | none |
| `PARTIAL_WITHDRAW_AND_REINVEST` | required | none | none | required | required |
| `WITHDRAW_ALL` | none | none | none | none | required |

A request with a missing required field, or a field the option doesn't use, is malformed (400).

**Notes:**

- Money fields are `NUMERIC(12,2)` in the database and `BigDecimal` in Java, never floating point.
- Members, ISAs, products, destination accounts and tax year summaries are **reference data** in phase 1, loaded by Flyway migrations as realistic test data. In a real building society they would come from other systems.
- The annual allowance and instruction window are **configuration** (R6), not tables.
- Only the maturity instruction table is written by the API.
- A database constraint allows at most one `ACTIVE` instruction per ISA (R2).
- Only the last four digits of account numbers are stored (data minimisation).

## 5. Endpoints

All paths are under `/v1/members/{memberId}`.

| # | Method and path | Purpose | Returns |
|---|---|---|---|
| 1 | `GET /isas` | the member's ISAs, so the app can show maturing ones | each ISA with its maturity date, expected balance and **instruction window** (open or not, opens at, closes at) |
| 2 | `GET /isas/{isaId}/reinvestment-products` | products the member can reinvest into | available products with name, type, term and minimum deposit |
| 3 | `GET /destination-accounts` | where withdrawn money can go | the member's society accounts and nominated bank account, masked |
| 4 | `GET /isas/{isaId}/available-to-deposit` | how much new money option 2 can add | the amount for the tax year containing the maturity date (R4) |
| 5 | `POST /isas/{isaId}/maturity-instructions` | submit or change an instruction | `201 Created` with the recorded instruction, or a rejection (section 6) |
| 6 | `GET /isas/{isaId}/maturity-instructions/active` | view the current instruction | the active instruction, or `404` if none |

- Endpoint 1 works out whether the window is open (R1), so the app never has to.
- Endpoint 5 is the only one that writes. Submitting again while the window is open replaces the active instruction (R2). It uses POST rather than PUT because a change creates a new record and supersedes the old one, instead of overwriting it.
- Endpoints 2, 3 and 4 only read reference data.

## 6. Error handling


| Situation | Status | Example `code` |
|---|---|---|
| Malformed request: missing or extra fields for the option, invalid values | 400 Bad Request | `INVALID_REQUEST` |
| ISA doesn't exist or doesn't belong to the member | 404 Not Found | `ISA_NOT_FOUND` |
| No active instruction (endpoint 6) | 404 Not Found | `NO_ACTIVE_INSTRUCTION` |
| Well-formed request that breaks a business rule | 422 Unprocessable Content | any rule code from section 3, for example `INSTRUCTION_WINDOW_CLOSED`, `WITHDRAWAL_EXCEEDS_BALANCE`, `EXCEEDS_AVAILABLE_TO_DEPOSIT` |
| Unexpected error | 500 Internal Server Error | `INTERNAL_ERROR`, with a generic message; full details logged server-side only |

Errors use the Problem Details format (RFC 9457, `application/problem+json`), with a `code` field the app can act on:

```json
{
  "type": "about:blank",
  "title": "Unprocessable Content",
  "status": 422,
  "detail": "The withdrawal amount is more than the expected maturity balance.",
  "code": "WITHDRAWAL_EXCEEDS_BALANCE"
}
```

- 400 means the app sent something malformed (a bug to fix); 422 means the request was fine but the rules say no (show the member the reason).
- If a request breaks several rules, only the first failing rule is returned, checked in the order R0 to R5. This is a deliberate simplification; a form-style API might return all failures at once.
- Rejections are logged with their reason code and the ISA ID, but no personal data.

## 7. Testing approach


- **Test first:** each rule in section 3 gets a failing test before it's implemented, confirmed to fail for the expected reason.
- **Unit tests** for the validation rules, with no Spring context or database, so they're fast. Each rule has passing cases, failing cases and boundaries. For example:
    - window: the day it opens, the day before it opens, 23:59:59 on the day before maturity (accepted), 00:00 on the maturity date (rejected)
    - available to deposit: exactly the limit (accepted) and 1p over (rejected)
    - partial withdrawal: the balance minus 1p (accepted) and the full balance (rejected, that's option 4)
    - tax year: an instruction submitted in March for an ISA maturing in April uses the April tax year
    - each option with a missing required field, and with a field it doesn't use (400)
- **Integration tests** (`@SpringBootTest` with a Testcontainers PostgreSQL) for each endpoint through HTTP: the success case, a 400, a 404 and at least one 422 per rule group.
- **R2 history:** changing an instruction supersedes the old one, and an invalid change leaves the active one in place.
- **Database constraint:** a second `ACTIVE` instruction for the same ISA is refused by the database itself.
- **CI:** GitHub Actions runs every test on every pull request.