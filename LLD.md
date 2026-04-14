# SmartSure — Low Level Design (LLD)

## 1. Shared Library Overview

### 1.1 SmartSure.Shared.Common

**BaseEntity** — base class for all EF-tracked entities in Admin and Claims services:
```
Id          : int (PK, auto-increment)
CreatedAt   : DateTime (UTC, default = now)
UpdatedAt   : DateTime? (nullable)
```

**Result / Result\<T\>** — uniform service return wrapper:
- `IsSuccess : bool`
- `Data : T?`
- `ErrorMessage : string?`

**PagedResult\<T\>** — uniform pagination wrapper:
- `Items : IEnumerable<T>`
- `TotalCount : int`
- `Page : int`
- `PageSize : int`

**Roles** constants: `"User"`, `"Admin"`

---

### 1.2 SmartSure.Shared.Contracts — Event Definitions

**UserEvents:**
```
UserRegisteredEvent  : UserId, FullName, Email, Role, CreatedAt
UserLoggedInEvent    : UserId, Email, LoggedInAt
UserRoleChangedEvent : UserId, NewRole, ChangedAt
```

**PolicyEvents:**
```
PolicyCreatedEvent   : PolicyId, PolicyNumber, UserId, CustomerName,
                       InsuranceType, PremiumAmount, Status, CreatedAt
PolicyCancelledEvent : PolicyId, UserId, Reason, CancelledAt
PremiumPaidEvent     : PolicyId, Amount, PaidAt
```

**ClaimEvents:**
```
ClaimSubmittedEvent     : ClaimId, PolicyId, UserId, PolicyNumber, ClaimNumber,
                          ClaimAmount, IncidentDate, OldStatus, NewStatus
ClaimStatusChangedEvent : ClaimId, PolicyId, OldStatus, NewStatus
ClaimApprovedEvent      : ClaimId, PolicyId, UserId, ApprovedAmount, Remarks
ClaimRejectedEvent      : ClaimId, PolicyId, UserId, Remarks
```

---

### 1.3 SmartSure.Shared.Infrastructure

**MassTransitExtensions.AddMassTransitWithRabbitMq()** — registers MassTransit with RabbitMQ transport.
- Reads `RabbitMQ:Host`, `RabbitMQ:Port`, `RabbitMQ:Username`, `RabbitMQ:Password` from config
- Uses kebab-case endpoint name formatter
- Accepts optional `Action<IBusRegistrationConfigurator>` for per-service consumer registration

**SerilogExtensions** — configures file + console sinks with daily rolling log files.

**MegaStorageService** — implements `IMegaStorageService` for MEGA cloud file upload/download.

---

### 1.4 SmartSure.Shared.Security

**JwtTokenGenerator** — implements `IJwtTokenGenerator`:
- Signs tokens with RSA-256 private key
- Standard claims: `sub` (UserId), `email`, `role`, `jti`
- Optional `purpose` claim for password-reset tokens (15 min expiry)
- Normal tokens: 1 hour expiry

**JwtSettings:**
```
PrivateKey  : string (RSA PEM, newlines from \n)
PublicKey   : string (RSA PEM, newlines from \n)
Issuer      : string
Audience    : string
```

---

## 2. Identity Service

**Port:** 5001 | **DB:** `SmartSure_IdentityDB`

### 2.1 Domain Entities

**User:**
```
UserId                  : Guid (PK)
FullName                : string
Email                   : string (unique)
Phone                   : string?
Address                 : string?
IsActive                : bool (default true)
IsEmailVerified         : bool (default false)
VerificationToken       : string?
VerificationTokenExpiry : DateTime?
CreatedAt               : DateTime
UpdatedAt               : DateTime?
DeletedAt               : DateTime?
UserRoles               : ICollection<UserRole>
ExternalLogins          : ICollection<ExternalLogin>
Passwords               : ICollection<Password>
```

**Role:**
```
RoleId    : int (PK)
Name      : string  ("User" | "Admin")
CreatedAt : DateTime
UserRoles : ICollection<UserRole>
```

**UserRole:**
```
Id         : int (PK)
UserId     : Guid (FK → User)
RoleId     : int (FK → Role)
AssignedAt : DateTime
```

**Password:**
```
Id                 : int (PK)
UserId             : Guid (FK → User)
PasswordHash       : string (BCrypt)
LastChangedAt      : DateTime
MustChangePassword : bool
```

**ExternalLogin:**
```
Id          : int (PK)
UserId      : Guid (FK → User)
Provider    : string  ("Google")
ProviderKey : string  (Google sub claim)
CreatedAt   : DateTime
```

**OtpRecord:**
```
Id        : int (PK)
Email     : string
HashedOtp : string (BCrypt of 6-digit code)
Expiry    : DateTime (10 min)
Attempts  : int (max 3 before lock)
CreatedAt : DateTime
```

---

### 2.2 Repository Interfaces

**IUserRepository:**
- `GetByIdAsync(Guid)` → `User?`
- `GetByEmailAsync(string)` → `User?`
- `GetByVerificationTokenAsync(string)` → `User?`
- `AddAsync(User)`
- `UpdateAsync(User)`
- `GetPagedAsync(page, pageSize, role?, isActive?)` → `PagedResult<User>`
- `ReplaceRoleAsync(Guid userId, int newRoleId)`

**IRoleRepository:**
- `GetByNameAsync(string)` → `Role?`

**IOtpRepository:**
- `GetByEmailAsync(string)` → `OtpRecord?`
- `AddAsync(OtpRecord)`
- `UpdateAsync(OtpRecord)`
- `DeleteAsync(OtpRecord)`

---

### 2.3 Service Interfaces

**IAuthService:**
- `RegisterAsync(RegisterDto)` → `Result`
- `LoginAsync(LoginDto)` → `Result<LoginResponseDto>`
- `LogoutAsync(Guid userId, string token)` → `Result`
- `VerifyEmailAsync(string token)` → `Result`
- `ResendVerificationEmailAsync(string email)` → `Result`
- `ForgotPasswordAsync(string email)` → `Result`
- `VerifyOtpAsync(VerifyOtpDto)` → `Result<string>` (returns reset JWT)
- `ResetPasswordAsync(ResetPasswordDto, string resetToken)` → `Result`
- `GetProfileAsync(Guid)` → `Result<UserProfileDto>`
- `UpdateProfileAsync(Guid, UpdateProfileDto)` → `Result`
- `ChangePasswordAsync(Guid, ChangePasswordDto)` → `Result`

**IAdminUserService:**
- `GetUsersAsync(page, pageSize, role?, isActive?)` → `PagedResult<User>`
- `AssignRoleAsync(Guid userId, string roleName)` → `Result`
- `RevokeRoleAsync(Guid userId, string roleName)` → `Result`

**IGoogleAuthService:**
- `GetGoogleConsentUrl()` → `string`
- `HandleCallbackAsync(string code)` → `Result<LoginResponseDto>`

**ITokenBlacklistService:**
- `BlacklistAsync(string token)`
- `IsBlacklistedAsync(string token)` → `bool`

**IOtpService:**
- `GenerateAndSendOtpAsync(string email)` → `Result`
- `VerifyOtpAsync(string email, string otp)` → `Result`

---

### 2.4 DTOs

| DTO | Fields |
|---|---|
| `RegisterDto` | Email, FullName, Password |
| `LoginDto` | Email, Password |
| `LoginResponseDto` | AccessToken, Email, FullName, Roles[] |
| `VerifyOtpDto` | Email, OtpCode |
| `ResetPasswordDto` | Email, NewPassword |
| `UpdateProfileDto` | FullName, Phone?, Address? |
| `UserProfileDto` | UserId, FullName, Email, Phone?, Address?, IsEmailVerified |
| `ChangePasswordDto` | CurrentPassword, NewPassword |
| `AssignRoleDto` | RoleName |

---

### 2.5 API Endpoints

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Register new user |
| POST | `/api/auth/login` | Public | Login, returns JWT |
| POST | `/api/auth/logout` | Bearer | Blacklist token |
| GET | `/api/auth/verify-email?token=` | Public | Verify email link |
| POST | `/api/auth/resend-verification?email=` | Public | Resend verification email |
| POST | `/api/auth/forgot-password?email=` | Public | Send OTP to email |
| POST | `/api/auth/verify-otp` | Public | Verify OTP, returns reset JWT |
| POST | `/api/auth/reset-password` | Reset JWT | Set new password |
| GET | `/api/auth/me` | Bearer | Get own profile |
| PUT | `/api/auth/me` | Bearer | Update own profile |
| PUT | `/api/auth/change-password` | Bearer | Change password |
| GET | `/api/auth/google` | Public | Redirect to Google OAuth |
| GET | `/api/auth/google/callback?code=` | Public | Handle Google callback |
| GET | `/api/auth/users` | Admin | List all users (paged) |
| PUT | `/api/auth/users/{userId}/roles` | Admin | Assign role to user |
| DELETE | `/api/auth/users/{userId}/roles/{roleName}` | Admin | Revoke role from user |

---

### 2.6 Events Published

| Event | Trigger |
|---|---|
| `UserRegisteredEvent` | Successful registration |
| `UserLoggedInEvent` | Successful login |
| `UserRoleChangedEvent` | Role assigned or revoked |

---

## 3. Policy Service

**Port:** 5152 | **DB:** `SmartSure_PolicyDB`

### 3.1 Domain Entities

**Policy:**
```
Id                 : Guid (PK)
PolicyNumber       : string (e.g. "POL-XXXXXXXX")
UserId             : Guid (FK → PolicyHolder)
InsuranceSubTypeId : int (FK → InsuranceSubType)
Status             : string  ("Pending" | "Active" | "Cancelled" | "Expired")
PremiumAmount      : decimal
StartDate          : DateTime?
EndDate            : DateTime?
CreatedAt          : DateTime
UpdatedAt          : DateTime?
InsuranceSubType   : InsuranceSubType? (nav)
HomeDetails        : HomeDetails? (nav)
VehicleDetails     : VehicleDetails? (nav)
Payments           : ICollection<PaymentRecord>
```

**InsuranceType:**
```
Id          : int (PK)
Name        : string  ("Vehicle Insurance" | "Home Insurance")
Description : string?
IsActive    : bool
CreatedAt   : DateTime
UpdatedAt   : DateTime?
SubTypes    : ICollection<InsuranceSubType>
```

**InsuranceSubType:**
```
Id              : int (PK)
InsuranceTypeId : int (FK → InsuranceType)
Name            : string
Description     : string?
BasePremium     : decimal
IsActive        : bool
CreatedAt       : DateTime
UpdatedAt       : DateTime?
```

**VehicleDetails:**
```
Id            : Guid (PK)
PolicyId      : Guid (FK → Policy)
Make          : string
Model         : string
Year          : int
Vin           : string
LicensePlate  : string
AnnualMileage : int
CreatedAt     : DateTime
UpdatedAt     : DateTime?
```

**HomeDetails:**
```
Id                : Guid (PK)
PolicyId          : Guid (FK → Policy)
PropertyAddress   : string
PropertyValue     : decimal
YearBuilt         : int
ConstructionType  : string?
HasSecuritySystem : bool
HasFireAlarm      : bool
CreatedAt         : DateTime
UpdatedAt         : DateTime?
```

**PaymentRecord:**
```
Id            : Guid (PK)
PolicyId      : Guid (FK → Policy)
Amount        : decimal
PaymentMethod : string  ("CreditCard" | "BankTransfer")
TransactionId : string
Status        : string  ("Completed" | "Failed" | "Pending")
PaidAt        : DateTime
```

**PolicyHolder** (mirrored from Identity via event):
```
UserId    : Guid (PK)
FullName  : string
Email     : string
CreatedAt : DateTime
UpdatedAt : DateTime
```

**PolicyDocument:**
```
Id          : Guid (PK)
PolicyId    : Guid (FK → Policy)
DocumentUrl : string (MEGA URL)
FileName    : string
ContentType : string
FileSize    : long
UploadedAt  : DateTime
```

---

### 3.2 Repository Interfaces

**IPolicyRepository:**
- `GetPoliciesByUserIdAsync(Guid, page, pageSize)` → `PagedResult<Policy>`
- `GetPolicyByIdAsync(Guid)` → `Policy?`
- `GetPolicyByIdAndUserIdAsync(Guid, Guid)` → `Policy?`
- `AddPolicyAsync(Policy)`
- `UpdatePolicyAsync(Policy)`
- `GetHomeDetailsAsync(Guid policyId)` → `HomeDetails?`
- `AddHomeDetailsAsync(HomeDetails)` / `UpdateHomeDetailsAsync(HomeDetails)`
- `GetVehicleDetailsAsync(Guid policyId)` → `VehicleDetails?`
- `AddVehicleDetailsAsync(VehicleDetails)` / `UpdateVehicleDetailsAsync(VehicleDetails)`
- `GetLatestPolicyDocumentAsync(Guid policyId)` → `PolicyDocument?`
- `AddPolicyDocumentAsync(PolicyDocument)` / `UpdatePolicyDocumentAsync(PolicyDocument)`
- `GetPolicyHolderAsync(Guid userId)` → `PolicyHolder?`
- `AddPolicyHolderAsync(PolicyHolder)`

**IInsuranceCatalogRepository:**
- `GetAllTypesAsync()` → `IEnumerable<InsuranceType>`
- `GetTypeByIdAsync(int)` → `InsuranceType?`
- `AddTypeAsync(InsuranceType)` / `UpdateTypeAsync(InsuranceType)`
- `GetSubTypesByTypeIdAsync(int)` → `IEnumerable<InsuranceSubType>`
- `AddSubTypeAsync(InsuranceSubType)` / `UpdateSubTypeAsync(InsuranceSubType)`

**IPaymentRepository:**
- `GetPaymentsByPolicyIdAsync(Guid, page, pageSize)` → `PagedResult<PaymentRecord>`
- `AddPaymentAsync(PaymentRecord)`

---

### 3.3 Service Interfaces

**IPolicyManagementService:**
- `GetPoliciesByUserIdAsync(Guid, page, pageSize)` → `PagedResult<PolicyDto>`
- `GetPolicyDetailAsync(Guid policyId, Guid userId)` → `Result<PolicyDetailDto>`
- `BuyPolicyAsync(Guid userId, BuyPolicyDto)` → `Result<Guid>`
- `CancelPolicyAsync(Guid policyId)` → `Result`
- `CalculatePremiumAsync(Guid policyId, Guid userId)` → `Result<decimal>`
- `GetPolicyDetailsDocumentAsync(Guid, Guid)` → `Result<PolicyDocumentDto>`
- `SavePolicyDetailsDocumentAsync(Guid, Guid, UploadDocumentDto)` → `Result`
- `UpdatePolicyDetailsDocumentAsync(Guid, Guid, UploadDocumentDto)` → `Result`

**IInsuranceCatalogService:**
- `GetAllTypesAsync()` → `IEnumerable<InsuranceTypeDto>`
- `GetTypeByIdAsync(int)` → `InsuranceTypeDto?`
- `CreateTypeAsync(CreateInsuranceTypeDto)` → `Result<InsuranceTypeDto>`
- `UpdateTypeAsync(int, UpdateInsuranceTypeDto)` → `Result`
- `SoftDeleteTypeAsync(int)` → `Result`
- `GetSubTypesByTypeIdAsync(int)` → `IEnumerable<InsuranceSubTypeDto>`
- `CreateSubTypeAsync(CreateInsuranceSubTypeDto)` → `Result<InsuranceSubTypeDto>`
- `UpdateSubTypeAsync(int, UpdateInsuranceSubTypeDto)` → `Result`
- `SoftDeleteSubTypeAsync(int)` → `Result`

**IPaymentService:**
- `RecordPaymentAsync(Guid policyId, Guid userId, CreatePaymentDto)` → `Result<PaymentDto>`
- `GetPaymentsAsync(Guid policyId, Guid userId, page, pageSize)` → `PagedResult<PaymentDto>`

---

### 3.4 API Endpoints

| Method | Route | Auth | Description |
|---|---|---|---|
| GET | `/api/policy/insurance-types` | Bearer | List all insurance types |
| GET | `/api/policy/insurance-types/{typeId}` | Bearer | Get type by ID |
| POST | `/api/policy/insurance-types` | Admin | Create insurance type |
| PUT | `/api/policy/insurance-types/{typeId}` | Admin | Update insurance type |
| DELETE | `/api/policy/insurance-types/{typeId}` | Admin | Soft-delete insurance type |
| GET | `/api/policy/insurance-types/{typeId}/subtypes` | Bearer | List subtypes for a type |
| POST | `/api/policy/insurance-subtypes` | Admin | Create subtype |
| PUT | `/api/policy/insurance-subtypes/{subTypeId}` | Admin | Update subtype |
| DELETE | `/api/policy/insurance-subtypes/{subTypeId}` | Admin | Soft-delete subtype |
| GET | `/api/policy/policies` | Bearer | Get own policies (paged) |
| GET | `/api/policy/policies/{policyId}` | Bearer | Get policy detail |
| POST | `/api/policy/policies` | Bearer | Buy a policy (direct, no payment) |
| PUT | `/api/policy/policies/{policyId}/cancel` | Admin | Cancel a policy |
| GET | `/api/policy/policies/{policyId}/details` | Bearer | Get policy document |
| POST | `/api/policy/policies/{policyId}/details` | Bearer | Upload policy document |
| PUT | `/api/policy/policies/{policyId}/details` | Bearer | Update policy document |
| GET | `/api/policy/policies/{policyId}/premium` | Bearer | Get premium amount |
| POST | `/api/policy/policies/{policyId}/payments` | Bearer | Record payment manually |
| GET | `/api/policy/policies/{policyId}/payments` | Bearer | Get payment history |
| POST | `/api/policy/policies/razorpay/create-order` | Bearer | Create Razorpay order |
| POST | `/api/policy/policies/razorpay/verify-payment` | Bearer | Verify payment + create policy |
| GET | `/api/dashboard` | Bearer | User dashboard (policies + claims) |

---

### 3.5 Events Published

| Event | Trigger |
|---|---|
| `PolicyCreatedEvent` | `BuyPolicyAsync` success |
| `PolicyCancelledEvent` | `CancelPolicyAsync` success |
| `PremiumPaidEvent` | `VerifyAndCompletePaymentAsync` success |

### 3.6 Events Consumed

| Event | Consumer | Action |
|---|---|---|
| `UserRegisteredEvent` | `UserRegisteredConsumer` | Upsert `PolicyHolder` record |

### 3.7 Razorpay Payment Flow

```
1. User clicks "Pay" on step 4 of Buy Policy wizard
2. Frontend calls POST /api/policy/policies/razorpay/create-order
   → Backend creates Razorpay order, returns { orderId, amount, currency, keyId }
3. Frontend opens Razorpay checkout popup
4. User completes payment (test: Netbanking → Success)
5. Razorpay returns { razorpay_order_id, razorpay_payment_id, razorpay_signature }
6. Frontend calls POST /api/policy/policies/razorpay/verify-payment
   → Backend verifies HMAC-SHA256 signature
   → Calls BuyPolicyAsync to create the policy
   → Records PaymentRecord with TransactionId = razorpay_payment_id
   → Publishes PolicyCreatedEvent + PremiumPaidEvent
7. Frontend redirects to /policies
```

**Signature Verification:**
```
payload  = razorpay_order_id + "|" + razorpay_payment_id
expected = HMAC-SHA256(payload, KeySecret)
```

---

## 4. Claims Service

**Port:** 5008 | **DB:** `SmartSure_ClaimsDB`

### 4.1 Domain Entities

**Claim:**
```
Id           : int (PK, BaseEntity)
UserId       : Guid
PolicyId     : Guid (FK → ValidPolicy.PolicyId)
IncidentDate : DateTime
Description  : string
ClaimAmount  : decimal
Status       : string  ("Draft" | "Submitted" | "Under Review" | "Approved" | "Rejected" | "Withdrawn")
ClaimNumber  : string  (e.g. "CLM-XXXXXXXX")
CreatedAt    : DateTime (BaseEntity)
UpdatedAt    : DateTime? (BaseEntity)
Documents    : ICollection<ClaimDocument>
History      : ICollection<ClaimHistory>
```

**ClaimHistory:**
```
Id             : int (PK, BaseEntity)
ClaimId        : int (FK → Claim)
PreviousStatus : string
NewStatus      : string
ChangedByUserId: Guid
Remarks        : string
ChangedAt      : DateTime
```

**ClaimDocument:**
```
Id           : int (PK, BaseEntity)
ClaimId      : int (FK → Claim)
DocumentType : string  ("PoliceReport" | "Photos" | "Invoice")
FileName     : string
FileUrl      : string (MEGA URL)
UploadedAt   : DateTime
```

**ValidPolicy** (mirrored from Policy service via event):
```
Id           : int (PK, BaseEntity)
PolicyId     : Guid (unique)
UserId       : Guid
PolicyNumber : string
CustomerName : string
Status       : string  ("Pending" | "Active" | "Cancelled")
```

---

### 4.2 Repository Interfaces

**IClaimRepository:**
- `GetClaimsByUserIdAsync(Guid, page, pageSize, status?)` → `IEnumerable<Claim>`
- `GetClaimByIdAsync(int)` → `Claim?`
- `GetClaimsByPolicyIdAsync(Guid)` → `IEnumerable<Claim>`
- `AddClaimAsync(Claim)`
- `UpdateClaimAsync(Claim)`
- `GetClaimSummaryAsync(Guid userId)` → `Dictionary<string, int>`
- `GetValidPolicyAsync(Guid policyId)` → `ValidPolicy?`
- `AddValidPolicyAsync(ValidPolicy)`
- `UpdateValidPolicyAsync(ValidPolicy)`

**IClaimHistoryRepository:**
- `GetByClaimIdAsync(int)` → `IEnumerable<ClaimHistory>`
- `AddAsync(ClaimHistory)`

**IClaimDocumentRepository:**
- `GetByClaimIdAsync(int)` → `IEnumerable<ClaimDocument>`
- `GetByIdAsync(int)` → `ClaimDocument?`
- `AddAsync(ClaimDocument)`
- `DeleteAsync(int)`

---

### 4.3 Service Interfaces

**IClaimManagementService:**
- `GetClaimsAsync(Guid userId, page, pageSize, status?)` → `PagedResult<ClaimDto>`
- `GetClaimByIdAsync(int)` → `ClaimDto?`
- `GetClaimsByPolicyIdAsync(Guid)` → `IEnumerable<ClaimDto>`
- `GetClaimHistoryAsync(int)` → `IEnumerable<ClaimHistoryDto>`
- `GetClaimSummaryAsync(Guid)` → `Dictionary<string, int>`
- `InitiateClaimAsync(Guid userId, CreateClaimDto)` → `Result<ClaimDto>`
- `UpdateDraftClaimAsync(int, Guid, UpdateClaimDto)` → `Result`
- `SubmitClaimAsync(int, Guid)` → `Result`
- `WithdrawClaimAsync(int, Guid)` → `Result`
- `ProcessClaimAsync(int, Guid adminId, bool approve, string remarks)` → `Result`

**IClaimDocumentService:**
- `GetDocumentsAsync(int claimId)` → `IEnumerable<ClaimDocumentDto>`
- `UploadDocumentAsync(int, Guid, UploadClaimDocumentDto, Stream, string fileName)` → `Result<ClaimDocumentDto>`
- `DeleteDocumentAsync(int claimId, int docId, Guid userId)` → `Result`

---

### 4.4 Business Rules in `InitiateClaimAsync`

1. Look up `ValidPolicy` by `PolicyId` — return error if not found
2. Reject if `ValidPolicy.Status == "Cancelled"`
3. Validate `ClaimAmount ≤ IDV` (IDV back-calculated from `PremiumAmount` stored in `ValidPolicy`)
4. Generate `ClaimNumber` = `"CLM-" + Guid.NewGuid().ToString("N")[..8].ToUpper()`
5. Save claim with `Status = "Submitted"`
6. Publish `ClaimSubmittedEvent`

---

### 4.5 API Endpoints

| Method | Route | Auth | Description |
|---|---|---|---|
| GET | `/api/claims` | Bearer | Get own claims (paged, filterable by status) |
| GET | `/api/claims/{claimId}` | Bearer | Get claim by ID |
| POST | `/api/claims` | Bearer | Initiate/submit a claim |
| PUT | `/api/claims/{claimId}` | Bearer | Update draft claim |
| PUT | `/api/claims/{claimId}/submit` | Bearer | Submit draft claim |
| PUT | `/api/claims/{claimId}/withdraw` | Bearer | Withdraw claim |
| GET | `/api/claims/by-policy/{policyId}` | Bearer | Get claims for a policy |
| GET | `/api/claims/summary` | Bearer | Get claim status counts |
| GET | `/api/claims/{claimId}/documents` | Bearer | List claim documents |
| POST | `/api/claims/{claimId}/documents` | Bearer | Upload document (multipart) |
| DELETE | `/api/claims/{claimId}/documents/{docId}` | Bearer | Delete document |

---

### 4.6 Events Published

| Event | Trigger |
|---|---|
| `ClaimSubmittedEvent` | Claim submitted |
| `ClaimStatusChangedEvent` | Any status change |
| `ClaimApprovedEvent` | Claim approved (via `ProcessClaimAsync`) |
| `ClaimRejectedEvent` | Claim rejected (via `ProcessClaimAsync`) |

### 4.7 Events Consumed

| Event | Consumer | Action |
|---|---|---|
| `PolicyCreatedEvent` | `PolicyCreatedConsumer` | Insert `ValidPolicy` record |
| `PolicyCancelledEvent` | `PolicyCancelledConsumer` | Set `ValidPolicy.Status = "Cancelled"` |
| `ClaimApprovedEvent` | `ClaimApprovedConsumer` | Update `Claim.Status = "Approved"` |
| `ClaimRejectedEvent` | `ClaimRejectedConsumer` | Update `Claim.Status = "Rejected"` |
| `ClaimStatusChangedEvent` | `ClaimStatusChangedConsumer` | Update `Claim.Status` to new value |

---

## 5. Admin Service

**Port:** 5113 | **DB:** `SmartSure_AdminDB`

### 5.1 Domain Entities

**AdminUser:**
```
Id        : int (PK, BaseEntity)
UserId    : Guid (unique, from Identity)
FullName  : string
Email     : string
Role      : string
IsActive  : bool
LastLogin : DateTime
CreatedAt : DateTime (BaseEntity)
UpdatedAt : DateTime? (BaseEntity)
```

**AdminPolicy:**
```
Id            : int (PK, BaseEntity)
PolicyId      : Guid (unique, from Policy service)
PolicyNumber  : string
CustomerName  : string
InsuranceType : string
PremiumAmount : decimal
Status        : string  ("Active" | "Cancelled" | "Expired")
CreatedAt     : DateTime (BaseEntity)
UpdatedAt     : DateTime? (BaseEntity)
```

**AdminClaim:**
```
Id           : int (PK, BaseEntity)
ClaimId      : int (from Claims service)
UserId       : Guid
PolicyId     : Guid
CustomerName : string
PolicyNumber : string
ClaimNumber  : string
ClaimAmount  : decimal
Status       : string  ("Submitted" | "Under Review" | "Approved" | "Rejected" | "Cancelled")
IncidentDate : DateTime
CreatedAt    : DateTime (BaseEntity)
UpdatedAt    : DateTime? (BaseEntity)
```

**AuditLog:**
```
Id         : int (PK, BaseEntity)
UserId     : Guid
Action     : string
EntityName : string
EntityId   : string
Details    : string
IpAddress  : string
CreatedAt  : DateTime (BaseEntity)
```

**Report:**
```
Id         : int (PK, BaseEntity)
Title      : string
Type       : string  ("claims" | "policies" | "revenue" | "audit")
FileUrl    : string
Status     : string  ("Pending" | "Completed" | "Failed")
Parameters : string (JSON)
CreatedAt  : DateTime (BaseEntity)
```

---

### 5.2 Repository Interface

**IAdminRepository\<T\>** (generic):
- `GetByIdAsync(Guid)` → `T?`
- `GetAllAsync()` → `IEnumerable<T>`
- `AddAsync(T)`
- `UpdateAsync(T)`
- `DeleteAsync(Guid)`

---

### 5.3 Service Interfaces

**IAdminClaimsService:**
- `GetClaimsAsync(status?, fromDate?, userId?, page, pageSize)` → `PagedResult<AdminClaimDto>`
- `GetClaimByIdAsync(int)` → `AdminClaimDto?`
- `MarkAsUnderReviewAsync(int claimId, string remarks)` → `bool`
- `ApproveClaimAsync(int claimId, string remarks)` → `bool`
- `RejectClaimAsync(int claimId, string reason)` → `bool`

**IAdminPolicyService:**
- `GetPoliciesAsync(searchTerm?, page, pageSize)` → `PagedResult<AdminPolicyDto>`

**IAdminUsersService:**
- `GetUsersAsync(searchTerm?, role?, page, pageSize)` → `PagedResult<AdminUserDto>`
- `GetUserByIdAsync(Guid)` → `AdminUserDto?`
- `SoftDeleteUserAsync(Guid)` → `bool`

**IAdminDashboardService:**
- `GetDashboardKpisAsync()` → `DashboardKpiDto`

**IAdminReportsService:**
- `GetReportsAsync(page, pageSize)` → `PagedResult<ReportDto>`
- `TriggerReportGenerationAsync(CreateReportRequest)` → `ReportDto`
- `GetReportByIdAsync(Guid)` → `ReportDto?`

**IAdminAuditLogService:**
- `GetAuditLogsAsync(entityType?, userId?, fromDate?, page, pageSize)` → `PagedResult<AuditLogDto>`
- `CreateAuditLogAsync(AuditLogDto)`
- `LogActionAsync(action, entityName, entityId, details)`

**IEmailService:**
- `SendEmailAsync(string to, string subject, string htmlBody)` → `Task`

---

### 5.4 API Endpoints

| Method | Route | Auth | Description |
|---|---|---|---|
| GET | `/api/admin/dashboard` | Admin | KPI summary |
| GET | `/api/admin/claims` | Admin | List claims (filterable) |
| GET | `/api/admin/claims/{claimId}` | Admin | Get claim detail |
| PUT | `/api/admin/claims/{claimId}/review` | Admin | Mark as Under Review |
| PUT | `/api/admin/claims/{claimId}/approve` | Admin | Approve claim |
| PUT | `/api/admin/claims/{claimId}/reject` | Admin | Reject claim |
| GET | `/api/admin/policies` | Admin | List policies (searchable) |
| GET | `/api/admin/users` | Admin | List users (searchable) |
| GET | `/api/admin/users/{userId}` | Admin | Get user detail |
| DELETE | `/api/admin/users/{userId}` | Admin | Soft-delete user |
| GET | `/api/admin/audit-logs` | Admin | List audit logs (filterable) |
| GET | `/api/admin/reports` | Admin | List reports |
| POST | `/api/admin/reports` | Admin | Trigger report generation |
| GET | `/api/admin/reports/{reportId}` | Admin | Get report by ID |
| POST | `/api/admin/reports/generate` | Admin | Generate PDF and stream back |

---

### 5.5 Events Consumed

| Event | Consumer | Action |
|---|---|---|
| `UserRegisteredEvent` | `UserRegisteredConsumer` | Insert `AdminUser` |
| `UserLoggedInEvent` | `UserLoggedInConsumer` | Update `AdminUser.LastLogin`, write `AuditLog` |
| `UserRoleChangedEvent` | `UserRoleChangedConsumer` | Update `AdminUser.Role` |
| `PolicyCreatedEvent` | `PolicyCreatedConsumer` | Insert `AdminPolicy` |
| `PolicyCancelledEvent` | `PolicyCancelledConsumer` | Update `AdminPolicy.Status = "Cancelled"`, send email to user |
| `ClaimSubmittedEvent` | `ClaimSubmittedConsumer` | Insert `AdminClaim` |
| `ClaimStatusChangedEvent` | `ClaimStatusChangedConsumer` | Update `AdminClaim.Status` |
| `ClaimApprovedEvent` | `ClaimApprovedConsumer` | Update `AdminClaim.Status = "Approved"`, send email to user |
| `ClaimRejectedEvent` | `ClaimRejectedConsumer` | Update `AdminClaim.Status = "Rejected"`, send email to user |

---

### 5.6 Email Notifications

Sent via `IEmailService` (MailKit, Gmail SMTP) from consumers:

| Trigger | Recipient | Subject |
|---|---|---|
| Claim Approved | Claim owner | "Your Claim Has Been Approved" |
| Claim Rejected | Claim owner | "Your Claim Has Been Rejected" |
| Policy Cancelled | Policy owner | "Your Policy Has Been Cancelled" |

Config keys in `.env`:
```
MailSettings__Host
MailSettings__Port
MailSettings__Username
MailSettings__Password
MailSettings__FromEmail
MailSettings__FromName
```

---

## 6. API Gateway

**Port:** 5083 | **Library:** Ocelot

### 6.1 Route Configuration (`ocelot.json`)

| Upstream Path Pattern | Downstream Service | Port |
|---|---|---|
| `/api/auth/**` | Identity API | 5001 |
| `/api/policy/**` | Policy API | 5152 |
| `/api/dashboard/**` | Policy API | 5152 |
| `/api/claims/**` | Claims API | 5008 |
| `/api/admin/**` | Admin API | 5113 |

- No JWT validation at gateway — each downstream service validates independently
- Rate limiting configured per route via Ocelot middleware
- Logging via Serilog

---

## 7. Database Schema Summary

### SmartSure_IdentityDB
| Table | Key Columns |
|---|---|
| `Users` | UserId (PK), Email, FullName, IsActive, IsEmailVerified, VerificationToken |
| `Roles` | RoleId (PK), Name |
| `UserRoles` | Id (PK), UserId (FK), RoleId (FK) |
| `Passwords` | Id (PK), UserId (FK), PasswordHash, MustChangePassword |
| `ExternalLogins` | Id (PK), UserId (FK), Provider, ProviderKey |
| `OtpRecords` | Id (PK), Email, HashedOtp, Expiry, Attempts |

### SmartSure_PolicyDB
| Table | Key Columns |
|---|---|
| `Policies` | Id (PK, Guid), PolicyNumber, UserId, InsuranceSubTypeId, Status, PremiumAmount |
| `InsuranceTypes` | Id (PK), Name, IsActive |
| `InsuranceSubTypes` | Id (PK), InsuranceTypeId (FK), Name, BasePremium, IsActive |
| `VehicleDetails` | Id (PK, Guid), PolicyId (FK), Make, Model, Year, Vin, LicensePlate, AnnualMileage |
| `HomeDetails` | Id (PK, Guid), PolicyId (FK), PropertyAddress, PropertyValue, YearBuilt, HasSecuritySystem |
| `PaymentRecords` | Id (PK, Guid), PolicyId (FK), Amount, PaymentMethod, TransactionId, Status |
| `PolicyHolders` | UserId (PK, Guid), FullName, Email |
| `PolicyDocuments` | Id (PK, Guid), PolicyId (FK), DocumentUrl, FileName, FileSize |

### SmartSure_ClaimsDB
| Table | Key Columns |
|---|---|
| `Claims` | Id (PK, int), UserId, PolicyId, ClaimNumber, ClaimAmount, Status, IncidentDate |
| `ClaimHistory` | Id (PK, int), ClaimId (FK), PreviousStatus, NewStatus, ChangedByUserId, Remarks |
| `ClaimDocuments` | Id (PK, int), ClaimId (FK), DocumentType, FileName, FileUrl |
| `ValidPolicies` | Id (PK, int), PolicyId (Guid, unique), UserId, PolicyNumber, CustomerName, Status |

### SmartSure_AdminDB
| Table | Key Columns |
|---|---|
| `AdminUsers` | Id (PK, int), UserId (Guid), FullName, Email, Role, IsActive, LastLogin |
| `AdminPolicies` | Id (PK, int), PolicyId (Guid), PolicyNumber, CustomerName, InsuranceType, PremiumAmount, Status |
| `AdminClaims` | Id (PK, int), ClaimId (int), UserId, PolicyId, ClaimNumber, ClaimAmount, Status |
| `AuditLogs` | Id (PK, int), UserId, Action, EntityName, EntityId, Details, IpAddress |
| `Reports` | Id (PK, int), Title, Type, FileUrl, Status, Parameters |

---

## 8. Frontend Structure

**Framework:** Angular 19 | **Port:** 4200 | **UI:** Bootstrap 5 + Bootstrap Icons

### 8.1 Angular Services

| Service | Responsibilities |
|---|---|
| `AuthService` | Login, register, logout, profile, token storage in `localStorage` |
| `PolicyService` | Insurance types, buy policy, list policies, cancel, payments |
| `ClaimService` | List claims, initiate claim, upload documents, withdraw |
| `DashboardService` | Fetch user dashboard stats |

### 8.2 HTTP Interceptors

| Interceptor | Behaviour |
|---|---|
| `AuthInterceptor` | Attaches `Authorization: Bearer <token>` to all outgoing requests |
| `ApiInterceptor` | Prefixes all requests with gateway base URL (`http://localhost:5083`) |

### 8.3 Route Guards

| Guard | Behaviour |
|---|---|
| `AuthGuard` | Redirects to `/login` if no valid JWT in `localStorage` |

### 8.4 Pages & Components

| Component | Route | Auth | Description |
|---|---|---|---|
| `LandingComponent` | `/` | Public | Marketing landing page |
| `LoginComponent` | `/login` | Public | Email/password login form |
| `RegisterComponent` | `/register` | Public | Registration form |
| `VerifyEmailComponent` | `/verify-email` | Public | Email verification result |
| `ForgotPasswordComponent` | `/forgot-password` | Public | OTP-based password reset (3 steps) |
| `DashboardComponent` | `/dashboard` | User | Stats, recent policies, recent claims |
| `PoliciesComponent` | `/policies` | User | Policy list with status filter, detail modal, cancel |
| `BuyPolicyComponent` | `/policies/buy` | User | 4-step wizard: Category → Plan → Details → Payment |
| `ClaimsComponent` | `/claims` | User | Claims list with tabs, IDV validation, submit modal |
| `ProfileComponent` | `/profile` | User | View/edit profile, change password |
| `ShellComponent` | (layout) | User | Sidebar nav, header, logout |
| `AdminManageClaimsComponent` | `/admin/claims` | Admin | Claims table, approve/reject/review actions |
| `AdminManagePoliciesComponent` | `/admin/policies` | Admin | Policies table, cancel action |
| `AdminManageUsersComponent` | `/admin/users` | Admin | Users table, deactivate action |
| `AdminManageInsuranceComponent` | `/admin/insurance` | Admin | Insurance types/subtypes CRUD |
| `AdminReportsComponent` | `/admin/reports` | Admin | Generate and download PDF reports |
| `AdminAuditLogsComponent` | `/admin/audit-logs` | Admin | Audit log viewer |

### 8.5 Buy Policy Wizard Steps

1. **Category** — select Insurance Type (Vehicle / Home)
2. **Plan** — select Insurance SubType, shows base premium
3. **Details** — vehicle form (make, model, year, VIN, plate, mileage) or home form (address, value, year built, construction type, security system, fire alarm) with live IDV preview
4. **Payment** — shows final premium, payment method selection, confirm purchase

### 8.6 IDV Calculation (Frontend Only)

**Vehicle:**
```
Depreciation by age: <1yr=5%, <2yr=15%, <3yr=20%, <4yr=30%, <5yr=40%, ≥5yr=50%
IDV = ListedPrice × (1 - depreciation)
Premium = IDV × 2%
High mileage surcharge (>15,000 km/yr): +20%
```

**Home:**
```
Depreciation by age: <5yr=0%, <10yr=10%, <20yr=20%, <30yr=30%, ≥30yr=40%
IDV = PropertyValue × (1 - depreciation)
Premium = IDV × 0.1%
Security system discount: -10%
```

**Back-calculation from stored premium:**
```
Vehicle IDV = PremiumAmount / 0.02
Home IDV   = PremiumAmount / 0.001
```

---

## 9. MassTransit Queue Architecture

MassTransit creates one exchange per event type (fanout) and one queue per consumer class. All services share the same RabbitMQ instance.

### Exchange → Queue Binding Map

| Exchange (Event) | Queue (Consumer) | Service |
|---|---|---|
| `user-registered` | `admin-user-registered` | Admin |
| `user-registered` | `policy-user-registered` | Policy |
| `user-logged-in` | `admin-user-logged-in` | Admin |
| `user-role-changed` | `admin-user-role-changed` | Admin |
| `policy-created` | `admin-policy-created` | Admin |
| `policy-created` | `claims-policy-created` | Claims |
| `policy-cancelled` | `admin-policy-cancelled` | Admin |
| `policy-cancelled` | `claims-policy-cancelled` | Claims |
| `claim-submitted` | `admin-claim-submitted` | Admin |
| `claim-status-changed` | `admin-claim-status-changed` | Admin |
| `claim-status-changed` | `claims-claim-status-changed` | Claims |
| `claim-approved` | `admin-claim-approved` | Admin |
| `claim-approved` | `claims-claim-approved` | Claims |
| `claim-rejected` | `admin-claim-rejected` | Admin |
| `claim-rejected` | `claims-claim-rejected` | Claims |

---

## 10. Security Implementation Details

### JWT Flow
1. Identity service generates RSA-256 JWT on login
2. Token contains: `sub` (UserId), `email`, `role`, `jti`, `exp` (1 hr)
3. All downstream services validate using the RSA public key (configured in `appsettings.json` / `.env`)
4. On logout, `jti` is added to in-memory blacklist (`ITokenBlacklistService`)
5. Password-reset tokens carry `purpose=password-reset` claim and 15-min expiry

### Secret Management
- All secrets stored in `.env` at repo root
- `DotNetEnv` loaded at startup with walk-up directory search
- `.env` is `.gitignore`d; `.env.example` committed with empty values
- ASP.NET Core config section mapping uses `__` double-underscore separator

### CORS
- All services allow only `http://localhost:4200`

---
