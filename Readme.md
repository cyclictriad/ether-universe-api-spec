# Trust Wallet dApp - Frontend API Specification (KYC-Enhanced)

**Purpose:** Define all API endpoints the frontend will consume  
**Base URL:** `https://api.yourapp.com/api/v1`  
**Authentication:** JWT Bearer Token (except auth endpoints)

---

## Important: KYC-First Authentication Model

**This application does NOT support traditional email/password signup.** All user accounts are created through wallet-based authentication combined with mandatory KYC verification.

**Authentication Flow:**
1. User connects wallet and signs authentication challenge
2. System checks if wallet address exists in database
3. If new user: Returns limited token with `kyc_required: true` status
4. User must submit KYC documents (ID card/passport) to create account
5. OCR automatically extracts user information (name, DOB, nationality, etc.)
6. Upon KYC approval, user gains full access to trading features

**Key Principle:** No transactions (swaps, transfers, trading) are allowed until KYC verification is completed and approved.

---

## Table of Contents
1. [Authentication & KYC](#authentication--kyc)
2. [KYC Management](#kyc-management)
3. [Wallet Operations](#wallet-operations)
4. [Transaction Management](#transaction-management)
5. [Trading Operations](#trading-operations)
6. [Token Information](#token-information)
7. [WebSocket Feeds](#websocket-feeds)
8. [Error Responses](#error-responses)
9. [Glossary](#glossary)

---

## Authentication & KYC

### 1. Request Challenge Message
**Endpoint:** `POST /auth/wallet/challenge`

**Request:**
```json
{
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "chain_id": 1
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

**Frontend Usage:**
```javascript
// 1. Get challenge
const { challenge_id, message } = await api.post('/auth/wallet/challenge', {
  wallet_address: address,
  chain_id: 1
});

// 2. User signs in wallet
const signature = await wallet.signMessage(message);
```

---

### 2. Verify Signature & Get Token
**Endpoint:** `POST /auth/wallet/verify`

**Request:**
```json
{
  "challenge_id": "chall_abc123xyz",
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "signature": "0x8f3d2c1b...",
  "chain_id": 1
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
    "full_name": "John Doe",
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
    "full_name": "John Doe",
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
    "full_name": null,
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
    "full_name": null,
    "created_at": "2025-10-01T12:00:00Z",
    "can_trade": false
  }
}
```


---

### 3. Get User Profile
**Endpoint:** `GET /auth/profile`  
**Headers:** `Authorization: Bearer <token>`

**Response (KYC Approved):**
```json
{
  "user_id": "usr_12345",
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "kyc_status": "approved",
  "kyc_verified_at": "2024-06-20T14:30:00Z",
  "full_name": "John Doe",
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
  "full_name": null,
  "created_at": "2025-10-01T12:00:00Z",
  "can_trade": false
}
```

---

### 4. Refresh Token
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

### 5. Logout
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

## KYC Management

### 6. Submit KYC Documents
**Endpoint:** `POST /kyc/submit`  
**Headers:** `Authorization: Bearer <token>`  
**Content-Type:** `multipart/form-data`

**Request:**
```
document_type: "passport" | "national_id" | "drivers_license"
document_front: [File] (JPEG/PNG, max 10MB)
document_back: [File] (JPEG/PNG, max 10MB) - Required for national_id and drivers_license
selfie_photo: [File] (JPEG/PNG, max 10MB) - Required for identity verification
```

**Response (Submitted Successfully):**
```json
{
  "kyc_submission_id": "kyc_sub_abc123",
  "status": "processing",
  "submitted_at": "2025-10-01T12:15:00Z",
  "estimated_processing_time": "2-24 hours",
  "ocr_extracted_data": {
    "full_name": "John Doe",
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

### 7. Get KYC Status
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
    "full_name": "John Doe",
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

### 8. Resubmit KYC (After Rejection)
**Endpoint:** `POST /kyc/resubmit`  
**Headers:** `Authorization: Bearer <token>`  
**Content-Type:** `multipart/form-data`

Same request/response format as `/kyc/submit`.

**Additional Response Field:**
```json
{
  "previous_rejection_reason": "Document image quality insufficient",
  "attempt_number": 2
}
```

---

### 9. Get KYC History
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

## Wallet Operations
## Wallet Operations

### 10. Get Wallet Balances
**Endpoint:** `GET /wallet/balances`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `chain_id` (optional): Filter by chain (1, 56, 137)
- `include_zero_balances` (optional): Default false

**Response:**
```json
{
  "wallet_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "balances": [
    {
      "chain_id": 1,
      "chain_name": "Ethereum",
      "native": {
        "symbol": "ETH",
        "balance": "2.456789012345678901",
        "balance_usd": "4913.58",
        "price_usd": "2000.00"
      },
      "tokens": [
        {
          "contract_address": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
          "symbol": "USDT",
          "name": "Tether USD",
          "decimals": 6,
          "balance": "5000.000000",
          "balance_usd": "5000.00",
          "price_usd": "1.00",
          "logo_url": "https://assets.trustwallet.com/.../logo.png"
        }
      ]
    }
  ],
  "total_value_usd": "9913.58",
  "last_updated": "2025-10-01T12:05:00Z"
}
```

---

### 11. Get Transaction History
**Endpoint:** `GET /wallet/transactions`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `chain_id` (required): 1, 56, or 137
- `limit` (optional): Default 50, max 100
- `offset` (optional): Default 0
- `type` (optional): `transfer`, `swap`, `approve`, `contract_interaction`

**Response:**
```json
{
  "transactions": [
    {
      "tx_hash": "0x1234567890abcdef...",
      "chain_id": 1,
      "type": "swap",
      "status": "confirmed",
      "from_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
      "to_address": "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D",
      "block_number": 18234567,
      "timestamp": "2025-10-01T11:45:00Z",
      "gas_used": "187234",
      "transaction_fee_usd": "11.23",
      "actions": [
        {
          "type": "token_swap",
          "token_in": {
            "symbol": "USDT",
            "amount": "1000.0"
          },
          "token_out": {
            "symbol": "WETH",
            "amount": "0.5"
          }
        }
      ],
      "explorer_url": "https://etherscan.io/tx/0x1234..."
    }
  ],
  "pagination": {
    "total": 347,
    "limit": 50,
    "offset": 0
  }
}
```

---

## Transaction Management

### 12. Prepare Transaction
**Endpoint:** `POST /transactions/prepare`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "chain_id": 1,
  "action": "swap",
  "parameters": {
    "token_in": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
    "token_out": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
    "amount_in": "1000000000",
    "slippage_tolerance": 0.5,
    "deadline_minutes": 20
  }
}
```

**Response:**
```json
{
  "transaction_id": "tx_prep_abc123",
  "unsigned_transaction": {
    "from": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "to": "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D",
    "data": "0x38ed173900000000...",
    "value": "0x0",
    "gasLimit": "0x30d40",
    "maxFeePerGas": "0x6fc23ac00",
    "maxPriorityFeePerGas": "0x77359400",
    "nonce": 45,
    "chainId": 1,
    "type": 2
  },
  "summary": {
    "action": "Swap 1000 USDT for ~0.5 WETH",
    "expected_output": "0.500000000000000000",
    "minimum_output": "0.497500000000000000",
    "price_impact": "0.08",
    "gas_estimate_usd": "12.00"
  },
  "expires_at": "2025-10-01T12:20:00Z"
}
```

**Frontend Usage:**
```javascript
// 1. Prepare transaction
const { transaction_id, unsigned_transaction } = await api.post('/transactions/prepare', {
  chain_id: 1,
  action: 'swap',
  parameters: { token_in, token_out, amount_in, slippage_tolerance: 0.5 }
});

// 2. User signs in wallet
const txHash = await wallet.sendTransaction(unsigned_transaction);

// 3. Submit to backend
await api.post('/transactions/submit', { transaction_id, tx_hash: txHash });
```

---

### 13. Check Token Allowance
**Endpoint:** `GET /transactions/check-allowance`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `chain_id` (required)
- `token_address` (required)
- `spender_address` (required)
- `amount` (required): Amount in smallest unit

**Response:**
```json
{
  "has_sufficient_allowance": false,
  "current_allowance": "0",
  "required_allowance": "1000000000",
  "needs_approval": true
}
```

---

### 14. Prepare Approval Transaction
**Endpoint:** `POST /transactions/prepare-approval`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "chain_id": 1,
  "token_address": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
  "spender_address": "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D",
  "amount": "unlimited"
}
```

**Response:**
```json
{
  "transaction_id": "tx_prep_approval_xyz",
  "unsigned_transaction": {
    "from": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "to": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
    "data": "0x095ea7b3000000...",
    "value": "0x0",
    "gasLimit": "0x10000",
    "maxFeePerGas": "0x6fc23ac00",
    "maxPriorityFeePerGas": "0x77359400",
    "nonce": 44,
    "chainId": 1,
    "type": 2
  },
  "summary": {
    "action": "Approve USDT spending",
    "amount": "Unlimited",
    "gas_estimate_usd": "3.90"
  },
  "expires_at": "2025-10-01T12:20:00Z"
}
```

---

### 15. Submit Transaction
**Endpoint:** `POST /transactions/submit`  
**Headers:** `Authorization: Bearer <token>`

**Request:**
```json
{
  "transaction_id": "tx_prep_abc123",
  "tx_hash": "0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba"
}
```

**Response:**
```json
{
  "transaction_id": "tx_prep_abc123",
  "tx_hash": "0x9876543210fedcba...",
  "status": "pending",
  "submitted_at": "2025-10-01T12:05:30Z",
  "explorer_url": "https://etherscan.io/tx/0x9876543210fedcba..."
}
```

---

### 16. Get Transaction Status
**Endpoint:** `GET /transactions/{tx_hash}/status`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `chain_id` (required)

**Response (Pending):**
```json
{
  "tx_hash": "0x9876543210fedcba...",
  "status": "pending",
  "submitted_at": "2025-10-01T12:05:30Z",
  "confirmations": 0,
  "estimated_confirmation_time": "20s"
}
```

**Response (Confirmed):**
```json
{
  "tx_hash": "0x9876543210fedcba...",
  "status": "confirmed",
  "submitted_at": "2025-10-01T12:05:30Z",
  "confirmed_at": "2025-10-01T12:06:15Z",
  "confirmations": 15,
  "block_number": 18234567,
  "gas_used": "187234",
  "transaction_fee_usd": "10.67",
  "execution_details": {
    "success": true,
    "token_in": "USDT",
    "amount_in": "1000.0",
    "token_out": "WETH",
    "amount_out": "0.500123456789012345"
  }
}
```

**Response (Failed):**
```json
{
  "tx_hash": "0x9876543210fedcba...",
  "status": "failed",
  "confirmed_at": "2025-10-01T12:06:15Z",
  "block_number": 18234567,
  "transaction_fee_usd": "2.57",
  "error": {
    "code": "EXECUTION_REVERTED",
    "message": "Slippage tolerance exceeded"
  }
}
```

---

## Trading Operations

### 17. Get Supported Trading Pairs
**Endpoint:** `GET /trading/pairs`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `chain_id` (optional): Filter by chain
- `search` (optional): Search by symbol

**Response:**
```json
{
  "pairs": [
    {
      "pair_id": "WETH_USDT_1",
      "chain_id": 1,
      "base_token": {
        "symbol": "WETH",
        "contract": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
        "decimals": 18
      },
      "quote_token": {
        "symbol": "USDT",
        "contract": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
        "decimals": 6
      },
      "liquidity_usd": "45678901.23",
      "volume_24h_usd": "12345678.90"
    }
  ]
}
```

---

### 18. Get Token Price
**Endpoint:** `GET /trading/price`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `token_in` (required): Contract address
- `token_out` (required): Contract address
- `chain_id` (required)
- `amount_in` (optional): For calculating price impact

**Response:**
```json
{
  "token_in": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
  "token_out": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
  "exchange_rate": "0.0005",
  "inverse_rate": "2000.0",
  "price_usd": {
    "token_in": "1.00",
    "token_out": "2000.00"
  },
  "price_impact": "0.08",
  "updated_at": "2025-10-01T12:05:45Z"
}
```

---

### 19. Get Order History
**Endpoint:** `GET /trading/orders`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `chain_id` (optional)
- `status` (optional): `pending`, `completed`, `failed`
- `limit` (optional): Default 50
- `offset` (optional)

**Response:**
```json
{
  "orders": [
    {
      "order_id": "ord_abc123",
      "tx_hash": "0x9876543210fedcba...",
      "chain_id": 1,
      "type": "swap",
      "status": "completed",
      "token_in": {
        "symbol": "USDT",
        "amount": "1000.0"
      },
      "token_out": {
        "symbol": "WETH",
        "amount": "0.500123456789012345"
      },
      "gas_fee_usd": "10.67",
      "created_at": "2025-10-01T12:05:00Z",
      "executed_at": "2025-10-01T12:06:15Z"
    }
  ],
  "pagination": {
    "total": 89,
    "limit": 50,
    "offset": 0
  }
}
```

---

## Token Information

### 20. Search Tokens
**Endpoint:** `GET /tokens/search`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `query` (required): Token symbol or name
- `chain_id` (optional): Filter by chain
- `limit` (optional): Default 20

**Response:**
```json
{
  "tokens": [
    {
      "contract_address": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
      "symbol": "USDT",
      "name": "Tether USD",
      "decimals": 6,
      "chain_id": 1,
      "logo_url": "https://assets.trustwallet.com/.../logo.png",
      "price_usd": "1.00",
      "is_verified": true
    }
  ]
}
```

---

### 21. Get Token Details
**Endpoint:** `GET /tokens/{contract_address}`  
**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**
- `chain_id` (required)

**Response:**
```json
{
  "contract_address": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
  "symbol": "USDT",
  "name": "Tether USD",
  "decimals": 6,
  "chain_id": 1,
  "logo_url": "https://assets.trustwallet.com/.../logo.png",
  "price_usd": "1.00",
  "price_change_24h": "0.02",
  "volume_24h_usd": "45678901234",
  "market_cap_usd": "83456789012",
  "is_verified": true
}
```

---

## WebSocket Feeds

### Connection
**WebSocket URL:** `wss://api.yourapp.com/ws/v1`

**Authentication:** Send JWT as first message
```json
{
  "action": "authenticate",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

###  Subscribe to Price Feed
**Subscribe Message:**
```json
{
  "action": "subscribe",
  "channel": "prices",
  "pairs": ["WETH_USDT_1", "WBNB_USDT_56"]
}
```

**Price Update Message:**
```json
{
  "channel": "prices",
  "pair_id": "WETH_USDT_1",
  "price": "2000.45",
  "change_24h": "2.34",
  "timestamp": "2025-10-01T12:05:50Z"
}
```

**Unsubscribe Message:**
```json
{
  "action": "unsubscribe",
  "channel": "prices",
  "pairs": ["WETH_USDT_1"]
}
```

---

### Subscribe to Transaction Updates
**Subscribe Message:**
```json
{
  "action": "subscribe",
  "channel": "transactions",
  "tx_hash": "0x9876543210fedcba..."
}
```

**Status Update Message:**
```json
{
  "channel": "transactions",
  "tx_hash": "0x9876543210fedcba...",
  "status": "confirmed",
  "confirmations": 12,
  "block_number": 18234567
}
```

---

## Error Responses

All errors follow this format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {}
  }
}
```

### Common Error Codes

| HTTP Status | Error Code | Meaning | Frontend Action |
|-------------|-----------|---------|-----------------|
| 400 | `INVALID_REQUEST` | Malformed request | Show validation error |
| 400 | `INSUFFICIENT_BALANCE` | Not enough funds | Show "Insufficient balance" |
| 400 | `INSUFFICIENT_LIQUIDITY` | Low DEX liquidity | Suggest smaller amount |
| 400 | `SLIPPAGE_EXCEEDED` | Price moved too much | Increase slippage or retry |
| 400 | `UNSUPPORTED_TOKEN` | Token not available | Show "Token not supported" |
| 401 | `INVALID_SIGNATURE` | Signature verification failed | Re-authenticate |
| 401 | `UNAUTHORIZED` | Missing/invalid token | Redirect to login |
| 404 | `NOT_FOUND` | Resource doesn't exist | Show 404 page |
| 409 | `DUPLICATE_REQUEST` | Already submitted | Show existing tx status |
| 410 | `EXPIRED` | Prepared tx expired | Prepare new transaction |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests | Show "Please wait" message |
| 500 | `INTERNAL_SERVER_ERROR` | Server error | Show error, suggest retry |
| 502 | `BLOCKCHAIN_RPC_ERROR` | Blockchain unavailable | Show "Network issue" |

### Error Response Examples

**Insufficient Balance:**
```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Insufficient balance for transaction",
    "details": {
      "required": "1000.0 USDT",
      "available": "500.0 USDT"
    }
  }
}
```

**Invalid Signature:**
```json
{
  "error": {
    "code": "INVALID_SIGNATURE",
    "message": "Signature verification failed",
    "details": {
      "expected_signer": "0x742d35Cc...",
      "recovered_signer": "0x123456..."
    }
  }
}
```

**Rate Limit:**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again in 30 seconds.",
    "details": {
      "retry_after": 30
    }
  }
}
```

---

## Glossary

| Term | Definition |
|------|------------|
| **dApp** | Decentralized Application - blockchain-based app where users control their own assets |
| **Non-Custodial** | User keeps control of private keys; platform never accesses them |
| **Wallet Address** | Unique identifier for blockchain account (e.g., `0x742d35Cc...`) |
| **Chain ID** | Identifier for blockchain network (1=Ethereum, 56=BSC, 137=Polygon) |
| **Token** | Digital asset on blockchain (ERC-20 standard) |
| **Contract Address** | Smart contract location on blockchain |
| **Gas** | Fee paid to process blockchain transactions |
| **Nonce** | Transaction counter ensuring correct order |
| **Slippage** | Maximum acceptable price change (e.g., 0.5% = 0.5) |
| **Wei** | Smallest unit of ETH (1 ETH = 1,000,000,000,000,000,000 wei) |
| **Decimals** | Number of decimal places for token (ETH=18, USDT=6) |
| **Approval** | Permission for contract to spend your tokens |
| **DEX** | Decentralized Exchange (e.g., Uniswap, PancakeSwap) |
| **Liquidity** | Available trading volume in a pair |
| **Price Impact** | How much your trade affects the price |
| **tx_hash** | Unique transaction identifier on blockchain |
| **Block Number** | Sequential number of block containing transaction |
| **Confirmations** | Number of blocks added after your transaction |
| **SIWE** | Sign-In With Ethereum - authentication using wallet signature |
| **JWT** | JSON Web Token - used for API authentication |

---

**End of Specification**