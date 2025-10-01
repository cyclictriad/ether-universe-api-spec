# Trust Wallet dApp - Frontend API Specification (KYC-Enhanced)

**Purpose:** Define all API endpoints the frontend will consume  
**Base URL:** `https://trade.cradlevoices.com/api/v1/`  
**Authentication:** JWT Bearer Token (except auth endpoints)

---

## Important: KYC-First Authentication Model

**This application does NOT support traditional email/password signup.** All user accounts are created through wallet-based authentication combined with mandatory KYC verification.

**Authentication Flow:**
1. User connects wallet and signs authentication challenge
2. System checks if wallet address exists in database
3. System initiates a user based on the wallet_id
4. If new user: Returns limited token with `kyc_required: true` status
5. User must submit KYC documents (ID card/passport) to create account
6. OCR automatically extracts user information (name, DOB, nationality, etc.)
7. Upon KYC approval, user gains full access to trading features

**Key Principle:** No transactions (orders, deposits) on live accounts are allowed until KYC verification is completed and approved. Demo accounts can be used without KYC.

---

## Table of Contents
1. [Authentication & KYC](#authentication--kyc)
2. [User Management](#user-management)
3. [Wallet Operations](#wallet-operations)
4. [Trading Operations](#trading-operations)
5. [Market Data](#market-data)
6. [Settings & Security](#settings--security)

---

## Authentication & KYC

### 1. Request Challenge Message
**Endpoint:** `POST /auth/wallet/challenge`

**Request:**
```json
{
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
}
```

**Response:**
```json
{
  "challenge_id": "chall_abc123xyz",
  "message": "Sign this message to authenticate:\n\nWallet: 0x742d35Cc...\nNonce: 8c7f2e1d\nTimestamp: 2025-10-01T12:00:00Z",
  "expires_at": "2025-10-01T12:10:00Z"
}
```

---

### 2. Verify Signature & Get Token
**Endpoint:** `POST /auth/wallet/verify`

**Request:**
```json
{
  "challenge_id": "chall_abc123xyz",
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "signature": "0x8f3d2c1b..."
}
```

**Response (Existing User - KYC Approved):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJlZnJlc2gifQ...",
  "expires_in": 3600,
  "user": {
    "user_id": "usr_12345",
    "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "is_new_user": false,
    "kyc_status": "approved",
    "kyc_verified_at": "2024-06-20T14:30:00Z",
    "first_name": "John",
    "last_name": "Doe",
    "created_at": "2024-06-15T10:30:00Z",
    "can_trade": true
  }
}
```

**Response (Existing User - KYC Pending):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJlZnJlc2gifQ...",
  "expires_in": 3600,
  "user": {
    "user_id": "usr_12345",
    "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "is_new_user": false,
    "kyc_status": "pending",
    "kyc_submitted_at": "2025-10-01T10:00:00Z",
    "first_name": null,
    "last_name": null,
    "created_at": "2025-09-28T08:15:00Z",
    "can_trade": false
  }
}
```

**Response (Existing User - KYC Rejected):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJlZnJlc2gifQ...",
  "expires_in": 3600,
  "user": {
    "user_id": "usr_12345",
    "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "is_new_user": false,
    "kyc_status": "rejected",
    "kyc_rejected_at": "2025-09-30T16:45:00Z",
    "rejection_reason": "Document image quality insufficient",
    "first_name": null,
    "last_name": null,
    "created_at": "2025-09-28T08:15:00Z",
    "can_trade": false,
    "can_resubmit": true
  }
}
```

**Response (New User - No KYC):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJlZnJlc2gifQ...",
  "expires_in": 3600,
  "user": {
    "user_id": "usr_67890",
    "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "is_new_user": true,
    "kyc_status": "not_submitted",
    "kyc_required": true,
    "first_name": null,
    "last_name": null,
    "created_at": "2025-10-01T12:00:00Z",
    "can_trade": false
  }
}
```

---

### 3. Refresh Token
**Endpoint:** `POST /auth/token/refresh`

**Request:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IlJlZnJlc2gifQ..."
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600
}
```

---

### 4. Logout
**Endpoint:** `POST /auth/logout`  
**Headers:** `Authorization: Bearer <token>`

**Request:** Empty body

**Response:**
```json
{
  "message": "Successfully logged out"
}
```

---

### 5. Submit KYC Documents
**Endpoint:** `POST /kyc/submit`  
**Headers:** `Authorization: Bearer <token>`  
**Content-Type:** `multipart/form-data`

**Request:**
```
document_type: "passport" | "national_id" | "drivers_license"
document_front: [File] (JPEG/PNG, max 10MB)
document_back: [File] (JPEG/PNG, max 10MB) - Required for national_id and drivers_license
```

**Response (Submitted Successfully):**
```json
{
  "kyc_submission_id": "kyc_sub_abc123",
  "status": "processing",
  "submitted_at": "2025-10-01T12:15:00Z",
  "estimated_processing_time": "2-24 hours",
  "ocr_extracted_data": {
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": "1990-05-15",
    "nationality": "US",
    "document_number": "P12345678",
    "document_expiry": "2030-05-14",
    "confidence_score": 0.95
  },
  "message": "Documents submitted successfully. We'll review your application and notify you via email."
}
```

**Response (OCR Failed - Manual Review Required):**
```json
{
  "kyc_submission_id": "kyc_sub_xyz789",
  "status": "manual_review",
  "submitted_at": "2025-10-01T12:15:00Z",
  "estimated_processing_time": "24-48 hours",
  "ocr_extracted_data": null,
  "message": "Documents received. Due to image quality, manual review is required."
}
```

---

### 6. Get KYC Status
**Endpoint:** `GET /kyc/status`  
**Headers:** `Authorization: Bearer <token>`

**Response (Pending):**
```json
{
  "kyc_submission_id": "kyc_sub_abc123",
  "status": "pending",
  "submitted_at": "2025-10-01T12:15:00Z",
  "processing_stage": "document_verification",
  "estimated_completion": "2025-10-01T18:00:00Z",
  "can_trade": false
}
```

**Response (Approved):**
```json
{
  "kyc_submission_id": "kyc_sub_abc123",
  "status": "approved",
  "submitted_at": "2025-10-01T12:15:00Z",
  "approved_at": "2025-10-01T16:30:00Z",
  "verified_data": {
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": "1990-05-15",
    "nationality": "US"
  },
  "can_trade": true
}
```

**Response (Rejected):**
```json
{
  "kyc_submission_id": "kyc_sub_abc123",
  "status": "rejected",
  "submitted_at": "2025-10-01T12:15:00Z",
  "rejected_at": "2025-10-01T16:30:00Z",
  "rejection_reason": "Document image quality insufficient. Please ensure image is clear and all text is readable.",
  "rejection_details": [
    "Document front image is blurry",
    "Expiry date not visible"
  ],
  "can_resubmit": true,
  "can_trade": false
}
```

**Response (No Submission):**
```json
{
  "status": "not_submitted",
  "message": "You haven't submitted KYC documents yet.",
  "can_trade": false
}
```

---

### 7. Get KYC History
**Endpoint:** `GET /kyc/history`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "submissions": [
    {
      "kyc_submission_id": "kyc_sub_abc123",
      "status": "rejected",
      "submitted_at": "2025-09-28T10:00:00Z",
      "processed_at": "2025-09-28T14:30:00Z",
      "rejection_reason": "Document expired"
    },
    {
      "kyc_submission_id": "kyc_sub_xyz789",
      "status": "approved",
      "submitted_at": "2025-10-01T12:15:00Z",
      "approved_at": "2025-10-01T16:30:00Z"
    }
  ]
}
```

---

## User Management

### 8. Get User Profile
**Endpoint:** `GET /user/profile`  
**Headers:** `Authorization: Bearer <token>`

**Response (KYC Approved):**
```json
{
  "user_id": "usr_12345",
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "kyc_status": "approved",
  "kyc_verified_at": "2024-06-20T14:30:00Z",
  "first_name": "John",
  "last_name": "Doe",
  "date_of_birth": "1990-05-15",
  "nationality": "US",
  "document_type": "passport",
  "document_number": "P12345678",
  "created_at": "2024-06-15T10:30:00Z",
  "can_trade": true
}
```

**Response (KYC Not Submitted):**
```json
{
  "user_id": "usr_67890",
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "kyc_status": "not_submitted",
  "kyc_required": true,
  "first_name": null,
  "last_name": null,
  "created_at": "2025-10-01T12:00:00Z",
  "can_trade": false
}
```

---

### 9. Update User Profile
**Endpoint:** `PUT /user/profile`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "first_name": "John",
  "last_name": "Doe"
}
```

**Response:**
```json
{
  "user_id": "usr_12345",
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "kyc_status": "approved",
  "kyc_verified_at": "2024-06-20T14:30:00Z",
  "first_name": "John",
  "last_name": "Doe",
  "date_of_birth": "1990-05-15",
  "nationality": "US",
  "document_type": "passport",
  "document_number": "P12345678",
  "created_at": "2024-06-15T10:30:00Z",
  "can_trade": true
}
```

---

### 10. Get Platform Activities
**Endpoint:** `GET /user/platform-activities`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "user_id": "usr_12345",
  "registered_on": "2024-01-01T00:00:00Z",
  "last_login": "2025-10-01T12:00:00Z",
  "last_deposit": {
    "amount": "100.00",
    "currency": "USDT",
    "timestamp": "2025-09-30T10:00:00Z"
  },
  "last_withdrawal": {
    "amount": "2000.00",
    "currency": "USDT",
    "timestamp": "2025-09-28T14:30:00Z"
  }
}
```

---

## Wallet Operations

### 11. Get Wallet Balances
**Endpoint:** `GET /wallet/balances`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `include_zero_balances` (optional): Default false

**Response:**
```json
{
  "wallets": [
    {
      "wallet_id": 24,
      "pair_id": 20,
      "symbol": "USDT",
      "name": "Tether USD",
      "decimals": 6,
      "balance": "5000.000000",
      "balance_usd": "5000.00",
      "price_usd": "1.00"
    },
    {
      "wallet_id": 25,
      "pair_id": 1,
      "symbol": "BTC",
      "name": "Bitcoin",
      "decimals": 8,
      "balance": "0.05000000",
      "balance_usd": "5000.00",
      "price_usd": "100000.00"
    }
  ],
  "total_balance_usd": "10000.00"
}
```

---

### 12. Get Wallet Addresses
**Endpoint:** `GET /wallet/get-wallets`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "wallets": [
    {
      "wallet_id": 1,
      "pair_id": 4,
      "symbol": "BTC",
      "address": "0xbfbff...",
      "network": "Bitcoin"
    },
    {
      "wallet_id": 2,
      "pair_id": 20,
      "symbol": "USDT",
      "address": "0xabc123...",
      "network": "Ethereum (ERC20)"
    }
  ]
}
```

---

### 13. Deposit to Wallet
**Endpoint:** `POST /wallet/deposit`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "wallet_id": 24,
  "amount": 3
}
```

**Response:**
```json
{
  "deposit_id": "dep_abc123",
  "wallet_id": 24,
  "pair_id": 20,
  "symbol": "USDT",
  "amount": "3.000000",
  "amount_usd": "3.00",
  "status": "completed",
  "created_at": "2025-10-01T12:30:00Z"
}
```

---

## Trading Operations

### 14. Get User Accounts
**Endpoint:** `GET /trading/accounts`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "accounts": [
    {
      "id": 1,
      "user_id": 1,
      "account_type": "demo",
      "balance": 10000,
      "equity": 10000,
      "margin": 0,
      "free_margin": 10000,
      "margin_level": 0,
      "is_active": true,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": 2,
      "user_id": 1,
      "account_type": "live",
      "balance": 5000,
      "equity": 5120,
      "margin": 500,
      "free_margin": 4620,
      "margin_level": 1024,
      "is_active": true,
      "created_at": "2024-06-20T00:00:00Z",
      "updated_at": "2025-10-01T12:00:00Z"
    }
  ]
}
```

---

### 15. Get Orders
**Endpoint:** `GET /trading/orders`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `account_id` (optional): Filter by account
- `status` (optional): Filter by status (open, completed, cancelled)

**Response:**
```json
{
  "orders": [
    {
      "id": 1,
      "user_id": 1,
      "account_id": 1,
      "pair_id": 4,
      "pair_symbol": "BTC/USDT",
      "type": "long",
      "amount": 0.1,
      "price": 50000,
      "total_value": 5000,
      "leverage": 1,
      "status": "completed",
      "profit_loss": 250.50,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

---

### 16. Create Order
**Endpoint:** `POST /trading/orders`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "account_id": 3,
  "pair_id": 1,
  "type": "long",
  "amount_usdt": 2000,
  "leverage": 1,
  "delivery_time": "30s",
  "price_range": 20
}
```

**Response:**
```json
{
  "id": 5,
  "user_id": 1,
  "account_id": 3,
  "pair_id": 1,
  "pair_symbol": "BTC/USDT",
  "type": "long",
  "amount_usdt": 2000,
  "leverage": 1,
  "entry_price": 100000.00,
  "delivery_time": "30s",
  "price_range": 20,
  "status": "open",
  "created_at": "2025-10-01T12:30:00Z"
}
```

---

### 17. Get Profit Statistics
**Endpoint:** `GET /trading/profit-statistics`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `account_id` (required): Account to get statistics for
- `start_date` (optional): ISO 8601 date
- `end_date` (optional): ISO 8601 date

**Response:**
```json
{
  "account_id": 3,
  "statistics": [
    [1727789022, 0],
    [1727789025, 10],
    [1727789032, -5],
    [1727789042, 15]
  ]
}
```

---

## Market Data

### 18. Get Trading Pairs
**Endpoint:** `GET /market/trading-pairs`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `category_id` (optional): Filter by category

**Response:**
```json
{
  "trading_pairs": [
    {
      "id": 1,
      "symbol": "BTC/USD",
      "base_asset": "BTC",
      "quote_asset": "USD",
      "name": "Bitcoin",
      "value_usd": 100000,
      "percentage_change": 2.34,
      "high_24h": 101500,
      "low_24h": 98000,
      "volume_24h": 125000000,
      "category_id": 1,
      "category": "Digital Assets",
      "logo_url": "https://assets.trustwallet.com/.../logo.png"
    },
    {
      "id": 2,
      "symbol": "ETH/USD",
      "base_asset": "ETH",
      "quote_asset": "USD",
      "name": "Ethereum",
      "value_usd": 3500,
      "percentage_change": -1.5,
      "high_24h": 3600,
      "low_24h": 3450,
      "volume_24h": 85000000,
      "category_id": 1,
      "category": "Digital Assets",
      "logo_url": "https://assets.trustwallet.com/.../logo.png"
    }
  ]
}
```

---

### 19. Get Price Data
**Endpoint:** `GET /market/price-data/{pair_id}`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `start_time` (required): Unix timestamp in seconds
- `end_time` (optional): Unix timestamp in seconds, default now
- `interval` (optional): `1m`, `5m`, `15m`, `1h`, `4h`, `1d` (default: `1h`)

**Response:**
```json
{
  "pair_id": 1,
  "symbol": "BTC/USD",
  "interval": "1h",
  "price_data": [
    [1727789022, 100000],
    [1727789025, 100010],
    [1727789032, 99995],
    [1727789042, 100050]
  ]
}
```

---

### 20. Get Market Overview
**Endpoint:** `GET /market/overview`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "total_market_cap": "2500000000000.00",
  "total_volume_24h": "125000000000.00",
  "market_change_24h": 1.8,
  "btc_dominance": 52.3,
  "top_gainers": [
    {
      "pair_id": 5,
      "symbol": "SOL/USD",
      "percentage_change": 8.5
    }
  ],
  "top_losers": [
    {
      "pair_id": 12,
      "symbol": "ADA/USD",
      "percentage_change": -5.2
    }
  ]
}
```

---

## Settings & Security

### 21. Get User Settings
**Endpoint:** `GET /settings`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "theme": "dark",
  "language": "en",
  "notifications": true,
  "email_alerts": true,
  "sms_alerts": false,
  "two_factor_auth": false,
  "risk_management": "high",
  "max_leverage": 20,
  "auto_close_trades": false
}
```

---

### 22. Update User Settings
**Endpoint:** `PUT /settings`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "theme": "dark",
  "language": "en",
  "notifications": true,
  "email_alerts": true,
  "sms_alerts": false,
  "two_factor_auth": false,
  "risk_management": "high",
  "max_leverage": 20,
  "auto_close_trades": false
}
```

**Response:**
```json
{
  "theme": "dark",
  "language": "en",
  "notifications": true,
  "email_alerts": true,
  "sms_alerts": false,
  "two_factor_auth": false,
  "risk_management": "high",
  "max_leverage": 20,
  "auto_close_trades": false
}
```

---

### 23. Get User Security
**Endpoint:** `GET /security`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "requirePin": false,
  "privacyMode": false
}
```

---

### 24. Update User Security
**Endpoint:** `PUT /security`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "requirePin": true,
  "privacyMode": false
}
```

**Response:**
```json
{
  "requirePin": true,
  "privacyMode": false
}
```

---

### 25. Get Current Leverage
**Endpoint:** `GET /leverage`  
**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "leverage": 10
}
```

---

### 26. Update Leverage
**Endpoint:** `PUT /leverage`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "leverage": 10
}
```

**Response:**
```json
{
  "leverage": 10
}
```

---

## Business Logic Notes

### Trading Model (CFD)
This platform operates as a **Contract for Difference (CFD)** trading platform:
- **Deposits** = Adding collateral to your wallet (e.g., deposit 1 BTC from Binance)
- **Orders** = Speculative positions on price movements (e.g., go long on BTC/USDT)
- Users don't buy actual crypto, they bet on price direction
- Profits/losses are settled in USDT equivalent

### Order Execution
- Users can trade any pair using USDT balance as collateral
- Example: Have 1000 USDT → Can open long/short position on BTC/USDT
- Margin required = Position Size / Leverage

### KYC Requirements
- **Demo accounts**: No KYC required, can trade immediately
- **Live accounts**: KYC approval mandatory for trading and deposits
- **Withdrawals**: KYC approval mandatory

---

**End of Specification**