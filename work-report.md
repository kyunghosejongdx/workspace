# 리팩토링 작업 내역

## 목차
1. [코드 구조 정리](#1-코드-구조-정리)
2. [결제 공통화](#2-결제-공통화)
   - [2-1. PGService — 결제 공통 메서드 추가](#2-1-pgservice--결제-공통-메서드-추가)
   - [2-2. PgClientPaymentRequestBatchService — processPayment() 적용](#2-2-pgclientpaymentrequestbatchservice--processpayment-적용)
   - [2-3. UsageB2CService — processPayment() 적용](#2-3-usageb2cservice--processpayment-적용)
   - [2-4. UsageB2BService — processPayment() 적용](#2-4-usageb2bservice--processpayment-적용)
   - [2-5. DailyCompanyPayBatchService — processPayment() 적용](#2-5-dailycompanypaybatchservice--processpayment-적용)
   - [2-6. BillingManualProcessor — UsageB2BService로 이동 및 processPayment() 적용](#2-6-billingmanualprocessor--usageb2bservice로-이동-및-processpayment-적용)
3. [누락부분 추가](#3-누락부분-추가)
   - [3-1. 일반 결제 (paymentRequest) — 결제 실패/통신오류 이력 저장 누락](#3-1-일반-결제-paymentrequest--결제-실패통신오류-이력-저장-누락)
   - [3-2. 위약금 결제 (paymentPenaltyCharge) — 결제 실패/통신오류 이력 저장 누락](#3-2-위약금-결제-paymentpenaltycharge--결제-실패통신오류-이력-저장-누락)
   - [3-3. 사용요금 결제 (paymentUsageCharge) — 결제 실패/통신오류 이력 저장 누락](#3-3-사용요금-결제-paymentusagecharge--결제-실패통신오류-이력-저장-누락)
   - [3-4. 일일 기업 과금 결제 / 수동 과금 결제 — 통신오류 이력 저장 누락](#3-4-일일-기업-과금-결제--수동-과금-결제--통신오류-이력-저장-누락)
4. [테스트](#4-테스트)
   - [4-1. 일반 결제 (paymentRequest)](#4-1-일반-결제-paymentrequest)
     - [4-1-1. UsageB2CService](#4-1-1-usageb2cservice)
     - [4-1-2. UsageB2BService](#4-1-2-usageb2bservice)
   - [4-2. 위약금 결제 (paymentPenaltyCharge)](#4-2-위약금-결제-paymentpenaltycharge)
     - [4-2-1. UsageB2CService - 위약금 결제](#4-2-1-usageb2cservice---위약금-결제)
     - [4-2-2. UsageB2BService - 위약금 결제](#4-2-2-usageb2bservice---위약금-결제)
   - [4-3. 사용요금 결제 (paymentUsageCharge)](#4-3-사용요금-결제-paymentusagecharge)
     - [4-3-1. UsageB2BService - 사용요금 결제](#4-3-1-usageb2bservice---사용요금-결제)
   - [4-4. 대상 기업 수동 과금](#4-4-대상-기업-수동-과금)
   - [4-5. 일일 기업 과금 배치](#4-5-일일-기업-과금-배치)
   - [4-6. 개인사용자 과금 배치 (PgClientPaymentRequestBatch)](#4-6-개인사용자-과금-배치-pgclientpaymentrequestbatch)

---

## 변경 요약

| 항목 | 변경 내용 |
|------|----------|
| 코드 구조 정리 | `UsageController`, `UsageService` 삭제, 엔드포인트 `UsageB2BController`로 통합 |
| PGService | `processPayment()`, `requestIniPay()`, `PaymentResult` 추가 |
| PgClientPaymentRequestBatchService | `insertPayment()` → `processPayment()` 리네임, `pgService.processPayment()` 위임, private `requestIniPay()` 삭제, 통신오류 케이스 저장 추가 |
| UsageB2CService | `iniApproval()` 직접 호출 2곳 → `processPayment()` 교체, 실패/통신오류 케이스 저장 추가 |
| UsageB2BService | `iniApproval()` 직접 호출 3곳 → `processPayment()` 교체, 실패/통신오류 케이스 저장 추가 |
| DailyCompanyPayBatchService | private `requestIniPay()` 삭제, `insertPayment()` → `processPayment()` 리네임, `pgService.processPayment()` 위임, 통신오류 케이스 저장 추가 |
| BillingManualService / BillingManualController | **삭제** — `BillingManualProcessor` inner class를 `UsageB2BService`로 이동, `manualRunByCompany()` 도 `UsageB2BService`로 이동, 엔드포인트는 `UsageB2BController`로 통합 |

---

## 1. 코드 구조 정리

### 배경
- `UsageService.java` — 내용 없는 빈 서비스 클래스, 참조 없음
- `UsageController.java` — `dailyCompanyPayBatch()` 단일 엔드포인트만 보유, 다른 곳에서 참조 없음
- 두 클래스 모두 삭제 후 엔드포인트는 `UsageB2BController`로 통합

### As-Is
| 항목 | 내용 |
|------|------|
| 파일 | `UsageController.java`, `UsageService.java` |
| 엔드포인트 | `GET /api/usage/dailyCompanyPayBatch` |
| 반환 | `void` (배치 실행만, 결과 없음) |

### To-Be
| 항목 | 내용 |
|------|------|
| 파일 | `UsageController.java`, `UsageService.java` **삭제** |
| 엔드포인트 | `GET /api/usage/b2b/dailyCompanyPayBatch` (경로 변경) |
| 반환 | `CommonResult` (배치 완료 후 성공 응답 반환) |

```java
// UsageB2BController.java 추가
@GetMapping("/dailyCompanyPayBatch")
@Operation(summary = "기업 과금 배치 수동 실행")
public CommonResult dailyCompanyPayBatch() {
    dailyCompanyPayBatchService.dailyCompanyPayBatch();
    return responseService.getSuccessResult();
}
```

> **URL 변경**: `/api/usage/dailyCompanyPayBatch` → `/api/usage/b2b/dailyCompanyPayBatch`

---

## 2. 결제 공통화

### 배경
- 각 서비스마다 reqMap 빌드 + `iniApproval()` 직접 호출 + `TbPayPayment` 저장 로직이 중복
- 결제 실패/통신오류 케이스에서 `TbPayPayment` 저장이 누락되는 케이스 존재
- `PGService`에 공통 메서드를 추가하고 각 서비스에서 위임하는 구조로 변경

---

### 2-1. PGService — 결제 공통 메서드 추가

### 대상 파일
- `common/service/PGService.java`

### As-Is
- `iniApproval()` 만 존재 (이니시스 호출만, TbPayPayment 저장 없음)
- 각 서비스에서 직접 reqMap 빌드 + `iniApproval()` 호출 + `TbPayPayment.builder()` 저장

### To-Be

#### `PaymentResult` 이너 클래스 추가
```java
@Getter
public static class PaymentResult {
    private final boolean success;
    private final String paymentId;   // tb_pay_payment PK
    private final String paymentDate; // yyyy-MM-dd HH:mm:ss (성공 시만)
    private final String payDate;     // yyyyMMdd raw (성공 시만, updatePenaltyPaymentInfo 등에서 사용)
    private final String tid;         // 이니시스 거래번호 (취소 시 필요)
}
```

#### `requestIniPay()` 추가 — 이니시스 요청 파라미터 공통화
```java
public Map<String, Object> requestIniPay(String goodName, String buyerName, String buyerEmail,
                                         String buyerTel, String price, String billkey) throws Exception
```

#### `processPayment()` 추가 — 결제 전체 흐름 공통화
```java
public PaymentResult processPayment(String subscribeUserId, String adminCardId, String billKey,
                                    String goodName, String buyerName, String buyerEmail,
                                    String buyerTel, String price) throws Exception
```

| 케이스 | TbPayPayment 저장 | 반환 |
|--------|------------------|------|
| 통신 오류 (isEmpty) | `resultCode="NTW_ERR"`, `resultMsg="통신오류"` | `success=false` |
| 결제 실패 (resultCode ≠ "00") | 이니시스 응답값 전체 저장 | `success=false` |
| 결제 성공 (resultCode = "00") | 이니시스 응답값 전체 저장 | `success=true`, `paymentId`, `paymentDate`, `payDate`, `tid` |

> `"NTW_ERR"` 사용 이유: 이니시스 resultCode는 2자리 숫자만 사용하므로 충돌 없음.

---

### 2-2. PgClientPaymentRequestBatchService — processPayment() 적용

### 대상 파일
- `domain/usage/service/PgClientPaymentRequestBatchService.java`

### As-Is

```java
// private requestIniPay() — pgService.requestIniPay() 위임
private Map<String, Object> requestIniPay(PgClientPaymentRequestBatchDto dto) throws Exception {
    return pgService.requestIniPay(
            dto.getGoodName(), dto.getBillmanagerNm(), dto.getBillmanagerEmail(),
            dto.getBillmanagerPhone(), String.valueOf(dto.getNormalPrice()), dto.getBillKey()
    );
}

// insertPayment() — requestIniPay() 호출 후 TbPayPayment 직접 저장
private PgClientPaymentRequestBatchDto insertPayment(PgClientPaymentRequestBatchDto dto) throws Exception {
    Map<String, Object> inicisResult = requestIniPay(dto);
    if (inicisResult.isEmpty()) {
        rstDto.setPaymentSuccessYn("N"); // TbPayPayment 저장 안 함
    } else {
        TbPayPayment payment = TbPayPayment.builder()
                .subscribeUserId(dto.getSubscribeUserId())
                .invoiceId(dto.getInvoiceId())
                .adminCardId(dto.getAdminCardId())
                .billKey(dto.getBillKey())
                .resultCode((String) inicisResult.get("resultCode"))
                .resultMsg((String) inicisResult.get("resultMsg"))
                .payDate((String) inicisResult.get("payDate"))
                .payTime((String) inicisResult.get("payTime"))
                .payAuthCode((String) inicisResult.get("payAuthCode"))
                .tid((String) inicisResult.get("tid"))
                .price((String) inicisResult.get("price"))
                .cardCode((String) inicisResult.get("cardCode"))
                .cardQuota((String) inicisResult.get("cardQuota"))
                .checkFlg((String) inicisResult.get("checkFlg"))
                .prtcCode((String) inicisResult.get("prtcCode"))
                .cardNumber((String) inicisResult.get("cardNumber"))
                .returnValue(objectMapper.writeValueAsString(inicisResult))
                .regdtm(LocalDateTime.now())
                .mdfdtm(LocalDateTime.now())
                .build();
        paymentRepository.save(payment);
    }
}
```

### 변경 사항 — invoiceId 제거

- As-Is에서 `TbPayPayment`에 `.invoiceId(dto.getInvoiceId())` 를 저장하고 있었으나, `processPayment()` 공통 메서드에 `invoiceId` 파라미터가 없어 해당 필드 제거
- `invoiceId`는 `TbPayPayment`에 더 이상 저장되지 않음

### To-Be

> **메서드 리네임**: `insertPayment()` → `processPayment()` (역할을 더 명확히 표현)

```java
// insertPayment() → processPayment() 로 리네임 + pgService.processPayment() 위임
// private requestIniPay() 삭제
private PgClientPaymentRequestBatchDto processPayment(PgClientPaymentRequestBatchDto dto) throws Exception {
    PGService.PaymentResult result = pgService.processPayment(
            dto.getSubscribeUserId(),               // PG결제 사용자 ID
            dto.getAdminCardId(),                   // 카드정보 ID
            dto.getBillKey(),                       // 이니시스 빌키
            dto.getGoodName(),                      // 상품명
            dto.getBillmanagerNm(),                 // 구매자명
            dto.getBillmanagerEmail(),              // 구매자 이메일
            dto.getBillmanagerPhone(),              // 구매자 전화번호
            String.valueOf(dto.getNormalPrice())    // 결제 금액 (원 단위)
    );
    PgClientPaymentRequestBatchDto rstDto = new PgClientPaymentRequestBatchDto();
    rstDto.setPaymentId(result.getPaymentId());
    rstDto.setPaymentDate(result.getPaymentDate());
    rstDto.setPaymentSuccessYn(result.isSuccess() ? "Y" : "N");
    return rstDto;
}
```

### 제거된 임포트 (PgClientPaymentRequestBatchService)
- `TbPayPayment`, `PaymentRepository` — processPayment 내부에서 처리
- `Map` — private requestIniPay() 삭제로 불필요

---

### 2-3. UsageB2CService — processPayment() 적용

### 대상 파일
- `domain/usage/service/UsageB2CService.java`

### As-Is — `paymentRequest()` payNow=Y

```java
// reqMap 수동 빌드
Map<String, Object> reqMap = new HashMap<>();
reqMap.put("goodName", rstOrderInfo.get().getGoodName());
reqMap.put("buyerName", rstOrderInfo.get().getBillManagerNm());
// ... 생략

// iniApproval 직접 호출
Map<String, Object> iniApprovalRstMap = pgService.iniApproval(reqMap);
if (iniApprovalRstMap.isEmpty()) {
    return responseService.getFailResult(500, "결제 실패");
}

// 성공 시만 TbPayPayment 저장
if (iniApprovalRstMap.get("resultCode").equals("00")) {
    TbPayPayment paymentInfo = TbPayPayment.builder()
            .subscribeUserId(...)
            // ... 20줄
            .build();
    TbPayPayment savedPaymentRst = paymentRepository.save(paymentInfo);
    // asset 처리
}
```

### To-Be — `paymentRequest()` payNow=Y

```java
String adminCardId = registCardInfo(request);

PGService.PaymentResult result = pgService.processPayment(
        rstOrderInfo.get().getSubscribeUserId(),
        adminCardId,
        pgService.billKeyDec(request.getBillKey()), // 복호화 후 전달
        rstOrderInfo.get().getGoodName(),
        rstOrderInfo.get().getBillManagerNm(),
        rstOrderInfo.get().getBillManagerEmail(),
        rstOrderInfo.get().getBillManagerPhone(),
        price
);

if (!result.isSuccess()) {
    return responseService.getFailResult(422, "결제 실패");
}

try {
    // asset 처리 (result.getPaymentId(), result.getPaymentDate() 사용)
} catch (Exception e) {
    // 결제 성공 후 asset 처리 실패 시 이니시스 취소
    cancelReqMap.put("tid", result.getTid());
    pgService.iniApprovalCancel(cancelReqMap);
    throw new Exception(e);
}
```

---

### As-Is — `paymentPenaltyCharge()`

```java
// reqMap 수동 빌드 + iniApproval 직접 호출
Map<String, Object> reqMap = new HashMap<>();
reqMap.put("goodName", "위약금");
// ...
Map<String, Object> iniApprovalRstMap = pgService.iniApproval(reqMap);

// 성공 시만 TbPayPayment 저장 (실패/통신오류 저장 안 함)
if (iniApprovalRstMap.get("resultCode").equals("00")) {
    TbPayPayment paymentInfo = TbPayPayment.builder()...build();
    TbPayPayment savedPaymentRst = paymentRepository.save(paymentInfo);
    usageRepository.updatePenaltyPaymentInfo(..., paymentInfo.getPayDate());
}
```

### To-Be — `paymentPenaltyCharge()`

```java
PGService.PaymentResult result = pgService.processPayment(
        request.getSubscribeUserId(),                                           // PG결제 사용자 ID
        subcribeUserInfo.getAdminCardId(),                                      // 카드정보 ID
        subcribeUserInfo.getBillKey(),                                          // 이니시스 빌키 (복호화된 값)
        "위약금",                                                               // 상품명
        Optional.ofNullable(subcribeUserInfo.getBuyerName()).orElse(" "),       // 구매자명 (KMS 복호화 후)
        Optional.ofNullable(subcribeUserInfo.getBuyerEmail()).orElse(" "),      // 구매자 이메일 (KMS 복호화 후)
        Optional.ofNullable(subcribeUserInfo.getBuyerTel()).orElse("1555-0066"),// 구매자 전화번호
        String.valueOf(request.getPenaltyCharge())                             // 위약금 금액 (원 단위)
);

if (!result.isSuccess()) {
    return responseService.getFailResult(422, "결제 실패");
}

usageRepository.updatePenaltyPaymentInfo(
        request.getSubscribeUserId(),
        result.getPaymentId(),
        result.getPayDate() // yyyyMMdd raw
);
```

### 제거된 임포트 (UsageB2CService)
- `TbPayPayment` — processPayment 내부에서 처리
- `SimpleDateFormat` — 불필요

---

### 2-4. UsageB2BService — processPayment() 적용

### 대상 파일
- `domain/usage/service/UsageB2BService.java`

### As-Is — 3개 메서드 모두 동일 패턴

```java
// paymentRequest (payNow=Y), paymentPenaltyCharge, paymentUsageCharge
Map<String, Object> reqMap = new HashMap<>();
reqMap.put("goodName", ...);
// ... reqMap 수동 빌드

Map<String, Object> iniApprovalRstMap = pgService.iniApproval(reqMap);
if (iniApprovalRstMap.isEmpty()) {
    return responseService.getFailResult(...);
}
if (iniApprovalRstMap.get("resultCode").equals("00")) {
    TbPayPayment paymentInfo = TbPayPayment.builder()...build();
    paymentRepository.save(paymentInfo);
    // asset 처리
}
```

### To-Be

```java
PGService.PaymentResult result = pgService.processPayment(
        subscribeUserId, adminCardId, billKey,
        goodName, buyerName, buyerEmail, buyerTel, price
);

if (!result.isSuccess()) {
    return responseService.getFailResult(422, "결제 실패");
}
// asset 처리 (result.getPaymentId(), result.getPaymentDate() 사용)
```

| 메서드 | 변경 내용 |
|--------|----------|
| `paymentRequest()` payNow=Y | reqMap 빌드 + iniApproval + TbPayPayment builder 제거, processPayment() 교체, result.getPaymentDate() 사용 |
| `paymentPenaltyCharge()` | 동일 패턴 교체 |
| `paymentUsageCharge()` | 동일 패턴 교체 |

### 제거된 임포트 (UsageB2BService)
- `TbPayPayment` — processPayment 내부에서 처리
- `SimpleDateFormat` — 불필요

---

### 2-5. DailyCompanyPayBatchService — processPayment() 적용

### 대상 파일
- `domain/usage/service/DailyCompanyPayBatchService.java`

### As-Is

```java
// private requestIniPay() — reqMap 수동 빌드 + iniApproval() 직접 호출
private Map<String, Object> requestIniPay(DailyCompanyPayBatchDto dto) throws Exception {
    SimpleDateFormat sf = new SimpleDateFormat("yyyyMMddHHmmss");
    Map<String, Object> req = new HashMap<>();
    req.put("goodName", dto.getGoodName());
    // ...
    return pgService.iniApproval(req);
}

// insertPayment() — requestIniPay() 호출 후 TbPayPayment 직접 저장
private DailyCompanyPayBatchDto insertPayment(DailyCompanyPayBatchDto dto) throws Exception {
    Map<String,Object> inicisResult = requestIniPay(dto);
    if (inicisResult.isEmpty()) {
        dto.setPaymentSuccessYn("N"); // TbPayPayment 저장 안 함
    } else {
        TbPayPayment payment = TbPayPayment.builder()...build();
        paymentRepository.save(payment);
    }
}
```

### To-Be

> **메서드 리네임**: `insertPayment()` → `processPayment()` (역할을 더 명확히 표현)

```java
// insertPayment() → processPayment() 로 리네임 + pgService.processPayment() 위임 (private requestIniPay() 삭제)
private DailyCompanyPayBatchDto processPayment(DailyCompanyPayBatchDto dto) throws Exception {
    PGService.PaymentResult result = pgService.processPayment(
            dto.getSubscribeUserId(),               // PG결제 사용자 ID
            dto.getAdminCardId(),                   // 카드정보 ID
            dto.getBillKey(),                       // 이니시스 빌키
            dto.getGoodName(),                      // 상품명
            dto.getBuyerName(),                     // 구매자명
            dto.getBuyerEmail(),                    // 구매자 이메일
            dto.getBuyerTel(),                      // 구매자 전화번호
            String.valueOf(dto.getPrice())          // 결제 금액 (원 단위)
    );
    DailyCompanyPayBatchDto rstDto = new DailyCompanyPayBatchDto();
    rstDto.setPaymentId(result.getPaymentId());
    rstDto.setPaymentDate(result.getPaymentDate());
    rstDto.setPaymentSuccessYn(result.isSuccess() ? "Y" : "N");
    return rstDto;
}
```

### 제거된 임포트 (DailyCompanyPayBatchService)
- `TbPayPayment`, `PaymentRepository` — processPayment 내부에서 처리
- `SimpleDateFormat`, `Date`, `InetAddress`, `HashMap`, `Map` — private requestIniPay() 삭제로 불필요

---

### 2-6. BillingManualProcessor — UsageB2BService로 이동 및 processPayment() 적용

### 대상 파일
- `domain/usage/service/BillingManualService.java` → **삭제**
- `domain/usage/controller/BillingManualController.java` → **삭제**
- `domain/usage/service/UsageB2BService.java` — `BillingManualProcessor` inner class 추가

### As-Is

```java
// BillingManualService.java
@Slf4j
@Service
@RequiredArgsConstructor
public class BillingManualService {

    public CommonResult manualRunByCompany(String companyId) { ... }

    @Component
    @RequiredArgsConstructor
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public static class BillingManualProcessor {
        private DailyCompanyPayBatchDto insertPayment(DailyCompanyPayBatchDto dto) throws Exception {
            // reqMap 수동 빌드 + iniApproval() 직접 호출 + TbPayPayment 직접 저장
        }
        // ...
    }
}

// BillingManualController.java
@PostMapping("/company-pay")
public CommonResult manualRunCompanyPay(@RequestBody DailyCompanyPayBatchDto request) {
    return billingManualService.manualRunByCompany(request.getCompanyId());
}
```

### To-Be

- `BillingManualService.java`, `BillingManualController.java` **삭제**
- `BillingManualProcessor` → `UsageB2BService`의 static inner class로 이동
- `manualRunByCompany()` → `UsageB2BService`로 이동
- 엔드포인트 → `UsageB2BController`로 통합 (`POST /api/usage/b2b/manual-billing/company`)

> **메서드 리네임**: `insertPayment()` → `processPayment()` (역할을 더 명확히 표현)

```java
// UsageB2BService.java에 추가된 static inner class
@Component
@RequiredArgsConstructor
@Transactional(propagation = Propagation.REQUIRES_NEW)
public static class BillingManualProcessor {
    // ...
    private DailyCompanyPayBatchDto processPayment(DailyCompanyPayBatchDto dto) throws Exception {
        PGService.PaymentResult result = pgService.processPayment(
                dto.getSubscribeUserId(), dto.getAdminCardId(), dto.getBillKey(),
                dto.getGoodName(), dto.getBuyerName(), dto.getBuyerEmail(),
                dto.getBuyerTel(), String.valueOf(dto.getPrice())
        );
        DailyCompanyPayBatchDto rstDto = new DailyCompanyPayBatchDto();
        rstDto.setPaymentId(result.getPaymentId());
        rstDto.setPaymentDate(result.getPaymentDate());
        rstDto.setPaymentSuccessYn(result.isSuccess() ? "Y" : "N");
        return rstDto;
    }
}
```

| 항목 | As-Is | To-Be |
|------|-------|-------|
| 파일 | `BillingManualService.java`, `BillingManualController.java` | **삭제** |
| 로직 위치 | `BillingManualService` outer + inner | `UsageB2BService` static inner |
| 엔드포인트 | `POST /api/billing/manual/company-pay` | `POST /api/usage/b2b/manual-billing/company` |

---

## 3. 누락부분 추가

### 3-1. 일반 결제 (paymentRequest) — 결제 실패/통신오류 이력 저장 누락

### 대상 파일
- `domain/usage/service/UsageB2CService.java`
- `domain/usage/service/UsageB2BService.java`

### As-Is
- `payNow = "Y"` 인 경우 (즉시 결제)에만 해당
- `resultCode = "00"` (성공) 인 경우에만 `TbPayPayment` 저장
- 결제 실패 또는 이니시스 통신 오류 발생 시 이력 저장 없이 에러 반환

```java
if (iniApprovalRstMap.get("resultCode").equals("00")) {
    // TbPayPayment 저장 (성공만)
    paymentRepository.save(paymentInfo);
} else {
    return responseService.getFailResult(422, "승인 처리 오류");
    // 실패 이력 저장 안 함
}
```

### To-Be
- `processPayment()` 공통화로 **결제 성공 / 실패 / 통신오류 모두 `TbPayPayment` 저장**

| 케이스 | As-Is | To-Be |
|--------|-------|-------|
| 결제 성공 (resultCode = "00") | 저장 O | 저장 O |
| 결제 실패 (resultCode ≠ "00") | 저장 X | 저장 O |
| 통신 오류 (이니시스 무응답) | 저장 X | 저장 O (`resultCode="NTW_ERR"`) |

---

### 3-2. 위약금 결제 (paymentPenaltyCharge) — 결제 실패/통신오류 이력 저장 누락

### 대상 파일
- `domain/usage/service/UsageB2CService.java`
- `domain/usage/service/UsageB2BService.java`

### As-Is
- `resultCode = "00"` (성공) 인 경우에만 `TbPayPayment` 저장
- 결제 실패 또는 이니시스 통신 오류 발생 시 이력 저장 없이 에러 반환

```java
if (iniApprovalRstMap.get("resultCode").equals("00")) {
    // TbPayPayment 저장 (성공만)
    paymentRepository.save(paymentInfo);
} else {
    return responseService.getFailResult(422, "승인 처리 오류");
    // 실패 이력 저장 안 함
}
```

### To-Be
- `processPayment()` 공통화로 **결제 성공 / 실패 / 통신오류 모두 `TbPayPayment` 저장**

| 케이스 | As-Is | To-Be |
|--------|-------|-------|
| 결제 성공 (resultCode = "00") | 저장 O | 저장 O |
| 결제 실패 (resultCode ≠ "00") | 저장 X | 저장 O |
| 통신 오류 (이니시스 무응답) | 저장 X | 저장 O (`resultCode="NTW_ERR"`) |

---

### 3-3. 사용요금 결제 (paymentUsageCharge) — 결제 실패/통신오류 이력 저장 누락

### 대상 파일
- `domain/usage/service/UsageB2BService.java`

### As-Is
- `resultCode = "00"` (성공) 인 경우에만 `TbPayPayment` 저장
- 결제 실패 또는 이니시스 통신 오류 발생 시 이력 저장 없이 에러 반환

```java
if (iniApprovalRstMap.get("resultCode").equals("00")) {
    // TbPayPayment 저장 (성공만)
    paymentRepository.save(paymentInfo);
} else {
    return responseService.getFailResult(422, "승인 처리 오류");
    // 실패 이력 저장 안 함
}
```

### To-Be
- `processPayment()` 공통화로 **결제 성공 / 실패 / 통신오류 모두 `TbPayPayment` 저장**

| 케이스 | As-Is | To-Be |
|--------|-------|-------|
| 결제 성공 (resultCode = "00") | 저장 O | 저장 O |
| 결제 실패 (resultCode ≠ "00") | 저장 X | 저장 O |
| 통신 오류 (이니시스 무응답) | 저장 X | 저장 O (`resultCode="NTW_ERR"`) |

---

### 3-4. 일일 기업 과금 결제 / 수동 과금 결제 — 통신오류 이력 저장 누락

### 대상 파일
- `domain/usage/service/DailyCompanyPayBatchService.java`
- `domain/usage/service/UsageB2BService.java` (`BillingManualProcessor` inner class)

### As-Is
- 결제 성공 / 실패는 모두 `TbPayPayment` 저장 (기존에 처리됨)
- 이니시스 통신 오류 (빈 map 반환) 시에는 이력 저장 없이 `paymentSuccessYn="N"` 처리

```java
if (inicisResult.isEmpty()) {
    dto.setPaymentSuccessYn("N"); // TbPayPayment 저장 안 함
} else {
    paymentRepository.save(payment); // 성공/실패 모두 저장
}
```

### To-Be
- `processPayment()` 공통화로 **통신오류 케이스도 `TbPayPayment` 저장**

| 케이스 | As-Is | To-Be |
|--------|-------|-------|
| 결제 성공 (resultCode = "00") | 저장 O | 저장 O |
| 결제 실패 (resultCode ≠ "00") | 저장 O | 저장 O |
| 통신 오류 (이니시스 무응답) | 저장 X | 저장 O (`resultCode="NTW_ERR"`) |

## 4. 테스트

### 사전 준비

1. 로컬 서버 실행 확인
2. Swagger로 API 호출 준비
   - Swagger UI: `http://localhost:8080/swagger-ui/index.html`
   - API Docs: `http://localhost:8080/v3/api-docs`
3. DB에서 테스트에 사용할 계정/기업 정보 확인
   - 빌키 보유 개인 계정 → `subscribeUserId`, `orderId`
   - 빌키 보유 기업 계정 → `subscribeUserId`, `orderId`
   - 과금 대상 기업 → `companyId`
4. 각 테스트 후 DB 확인 쿼리:
   ```sql
   SELECT * FROM tb_pay_payment ORDER BY regdtm DESC LIMIT 10;
   ```

---

### 4-1. 일반 결제 (paymentRequest)

#### 4-1-1. UsageB2CService

**대상**: `UsageB2CService.paymentRequest()`

**▶ 테스트 플로우**

**1. 주문 생성** — `POST /api/usage/b2c/order/request`
```json
{
  "subscribeUserId": "{{subscribeUserId}}",
  "planType": "{{planType}}",
  "adminNm": "테스트",
  "email": "test@test.com",
  "mobNo": "01000000000",
  "normalPrice": 10000,
  "returnUrl": "http://localhost:8080"
}
```
> 응답 `data.orderId` 메모

**2. 결제 요청** — `POST /api/usage/b2c/payment/request`
```json
{
  "orderId": "{{orderId}}",
  "billKey": "{{billKey}}",
  "payNow": "Y"
}
```

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "msg": "성공",
    "paymentId": "f84b846f-b2ae-4b66-a597-02c1785d7842",
    "adminCardId": "2e353034-b4ed-48ea-af12-418d64bc3969"
  }
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : f7e9c3eb-d00a-4488-bf0c-17148b5dd298
subscribe_user_id : ff37a9d8-00bf-11f1-a03c-027e3f3436a9
admin_card_id : f7195414-6921-49b9-86b5-c4aab6e9b01a
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260624
pay_auth_code : 00299510
tid           : INIAPICARDINIBillTst20260624135620552191
price         : 25300
regdtm        : 2026-06-24 13:57:07
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 64b2a588-bfcd-408b-807c-fd041ca89e1e
subscribe_user_id : ff37a9d8-00bf-11f1-a03c-027e3f3436a9
admin_card_id : b6635b67-9d5a-4694-b835-fef8021b9425
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 01
result_msg    : [1113][실시간빌링실패|요청금액 0원 이하]
pay_date      : 20260624
tid           : INIAPICARDINIBillTst20260624140037411144
price         : 0
regdtm        : 2026-06-24 14:01:51
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : a827bbb4-f4f2-4e1e-aa5a-2101da142c5b
subscribe_user_id : ff37a9d8-00bf-11f1-a03c-027e3f3436a9
admin_card_id : 2ae93ca2-2ace-4a21-9dfe-01a5a6b31928
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-24 14:11:33
```

---

**테스트 결과**

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| 결제 성공 | `code=200` | `00` | ✓ |
| 결제 실패 | `code=422` | `01` | ✓ |
| 통신 오류 | `code=422` | `NTW_ERR` | ✓ |

---

#### 4-1-2. UsageB2BService

**대상**: `UsageB2BService.paymentRequest()`

**▶ 테스트 플로우**

**1. 주문 생성** — `POST /api/usage/b2b/order/request`
```json
{
  "subscribeUserId": "{{subscribeUserId}}",
  "planType": "{{planType}}",
  "adminNm": "테스트",
  "email": "test@test.com",
  "mobNo": "01000000000",
  "normalPrice": 10000,
  "returnUrl": "http://localhost:8080"
}
```
> 응답 `data.orderId` 메모

**2. 결제 요청** — `POST /api/usage/b2b/payment/request`
```json
{
  "orderId": "{{orderId}}",
  "billKey": "{{billKey}}",
  "payNow": "Y"
}
```

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "msg": "성공",
    "paymentId": "58709e78-b2ba-4ecd-b10e-36f9782ba828",
    "adminCardId": "dc6ddd41-96e2-4ce9-bbfd-75bfaef2b411"
  }
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 58709e78-b2ba-4ecd-b10e-36f9782ba828
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : dc6ddd41-96e2-4ce9-bbfd-75bfaef2b411
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260624
pay_auth_code : 00211553
tid           : INIAPICARDINIBillTst20260624162317452142
price         : 27500
regdtm        : 2026-06-24 16:24:13
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 93b75e7a-2d9d-4e5c-9add-f767c0c0547f
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : f7d547ed-2638-465c-af30-cf7817d0dd1d
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 01
result_msg    : [1113][실시간빌링실패|요청금액 0원 이하]
pay_date      : 20260624
pay_auth_code : ERROR
tid           : INIAPICARDINIBillTst20260624164126093287
price         : 0
regdtm        : 2026-06-24 16:41:46
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : dfd167b0-3511-4721-882b-6fbf0b51296f
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : 0cd9a6f5-0760-4108-9753-ae3592bf4926
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-24 16:43:46
```

---

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| A. 결제 성공 | `code=200` | `00` | ✓ |
| B. 결제 실패 | `code=422` | `01` | ✓ |
| C. 통신 오류 | `code=422` | `NTW_ERR` | ✓ |

---

### 4-2. 위약금 결제 (paymentPenaltyCharge)

#### 4-2-1. UsageB2CService - 위약금 결제

**대상**: `UsageB2CService.paymentPenaltyCharge()`

**▶ 테스트 플로우**

**1. 결제 요청** — `POST /api/usage/b2c/payment/penaltyCharge`
```json
{
  "subscribeUserId": "{{subscribeUserId}}",
  "penaltyCharge": 10000
}
```

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "msg": "성공"
  }
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 9efb08a4-f940-420a-b812-d0fdee4a3203
subscribe_user_id : ff37a9d8-00bf-11f1-a03c-027e3f3436a9
admin_card_id : 2ae93ca2-2ace-4a21-9dfe-01a5a6b31928
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260624
pay_auth_code : 00543733
tid           : INIAPICARDINIBillTst20260624165823433243
price         : 8000
regdtm        : 2026-06-24 16:58:46
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : dc34d713-4c32-4040-af74-d1e7710a41dc
subscribe_user_id : ff37a9d8-00bf-11f1-a03c-027e3f3436a9
admin_card_id : 2ae93ca2-2ace-4a21-9dfe-01a5a6b31928
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : ER0102
result_msg    : 파라미터 값이 잘못 되었습니다.
return_value  : {"resultMsg": "파라미터 값이 잘못 되었습니다.", "resultCode": "ER0102", "errorDetails": [{"field": "buyerName", "contents": "공백일 수 없습니다"}, {"field": "buyerEmail", "contents": "공백일 수 없습니다"}]}
regdtm        : 2026-06-24 16:55:06
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 93c3e7f9-c6d2-4133-a5c3-139593be9e13
subscribe_user_id : ff37a9d8-00bf-11f1-a03c-027e3f3436a9
admin_card_id : 2ae93ca2-2ace-4a21-9dfe-01a5a6b31928
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-26 11:19:52
```

---

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| A. 결제 성공 | `code=200` | `00` | ✓ |
| B. 결제 실패 | `code=422` | `ER0102` | ✓ |
| C. 통신 오류 | `code=422` | `NTW_ERR` | ✓ |

---

#### 4-2-2. UsageB2BService - 위약금 결제

**대상**: `UsageB2BService.paymentPenaltyCharge()`

**▶ 테스트 플로우**

**1. 결제 요청** — `POST /api/usage/b2b/payment/penaltyCharge`
```json
{
  "subscribeUserId": "{{subscribeUserId}}",
  "penaltyCharge": 10000
}
```

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "msg": "성공",
    "paymentId": "135584fb-615c-4e9f-bcd9-16ea68054e5e",
    "penaltyCharge": 8000
  }
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 135584fb-615c-4e9f-bcd9-16ea68054e5e
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : 0cd9a6f5-0760-4108-9753-ae3592bf4926
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260626
pay_auth_code : 00156261
tid           : INIAPICARDINIBillTst20260626113519281103
price         : 8000
regdtm        : 2026-06-26 11:35:56
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : c60b7eaf-c0b1-49f2-bb76-29b73c5fbe3c
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : 0cd9a6f5-0760-4108-9753-ae3592bf4926
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 01
result_msg    : [1113][실시간빌링실패|요청금액 0원 이하]
pay_date      : 20260626
pay_auth_code : ERROR
tid           : INIAPICARDINIBillTst20260626113704303146
price         : 0
regdtm        : 2026-06-26 11:37:23
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 7d7150b0-3095-4d9c-982c-cee320c3b198
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : 0cd9a6f5-0760-4108-9753-ae3592bf4926
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-26 11:29:56
```

---

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| A. 결제 성공 | `code=200` | `00` | ✓ |
| B. 결제 실패 | `code=422` | `01` | ✓ |
| C. 통신 오류 | `code=422` | `NTW_ERR` | ✓ |

---

### 4-3. 사용요금 결제 (paymentUsageCharge)

**대상**: `UsageB2BService.paymentUsageCharge()`

**▶ 테스트 플로우**

**1. 결제 요청** — `POST /api/usage/b2b/payment/usageCharge`
```json
{
  "subscribeUserId": "{{subscribeUserId}}"
}
```

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "msg": "성공",
    "paymentId": "5217eb81-1f0e-4685-8f09-dc45bda53154",
    "usageCharge": 6000
  }
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : 5217eb81-1f0e-4685-8f09-dc45bda53154
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : 0cd9a6f5-0760-4108-9753-ae3592bf4926
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260626
pay_auth_code : 00413453
tid           : INIAPICARDINIBillTst20260626134011733133
price         : 6000
regdtm        : 2026-06-26 13:40:29
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : c60b7eaf-c0b1-49f2-bb76-29b73c5fbe3c
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : 0cd9a6f5-0760-4108-9753-ae3592bf4926
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 01
result_msg    : [1113][실시간빌링실패|요청금액 0원 이하]
pay_date      : 20260626
pay_auth_code : ERROR
tid           : INIAPICARDINIBillTst20260626113704303146
price         : 0
regdtm        : 2026-06-26 11:37:23
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 422,
  "msg": "결제 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : eb50416f-5338-4717-9456-a704abf4c1cd
subscribe_user_id : 99b7b6b4-0265-11f1-a03c-027e3f3436a9
admin_card_id : 0cd9a6f5-0760-4108-9753-ae3592bf4926
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-26 13:43:47
```

---

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| A. 결제 성공 | `code=200` | `00` | ✓ |
| B. 결제 실패 | `code=422` | `01` | ✓ |
| C. 통신 오류 | `code=422` | `NTW_ERR` | ✓ |

---

### 4-4. 대상 기업 수동 과금

**대상**: `UsageB2BService.BillingManualProcessor`

**▶ 테스트 플로우**

**1. 결제 요청** — `POST /api/usage/b2b/manual-billing/company`
```json
{
  "companyId": "{{companyId}}"
}
```

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : b51f2d97-6448-4144-b4fd-595ca3aac834
subscribe_user_id : a598e6b2-4392-11f1-963e-7a06ad5292e5
admin_card_id : ddc58e38-4396-11f1-963e-7a06ad5292e5
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260626
pay_auth_code : 00064874
tid           : INIAPICARDINIBillTst20260626164547101218
price         : 1100
regdtm        : 2026-06-26 16:46:22
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 422,
  "msg": "과금 처리 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : b8892462-2046-488c-9479-b51b91c6d1c0
subscribe_user_id : a598e6b2-4392-11f1-963e-7a06ad5292e5
admin_card_id : ddc58e38-4396-11f1-963e-7a06ad5292e5
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 01
result_msg    : [1113][실시간빌링실패|요청금액 0원 이하]
pay_date      : 20260626
pay_auth_code : ERROR
tid           : INIAPICARDINIBillTst20260626163746571118
price         : 0
regdtm        : 2026-06-26 16:38:33
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 422,
  "msg": "과금 처리 실패"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : cf9b6b06-24be-46b3-a907-fdf51c95a640
subscribe_user_id : a598e6b2-4392-11f1-963e-7a06ad5292e5
admin_card_id : ddc58e38-4396-11f1-963e-7a06ad5292e5
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-26 16:36:17
```

---

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| A. 결제 성공 | `code=200` | `00` | ✓ |
| B. 결제 실패 | `code=422` | `01` | ✓ |
| C. 통신 오류 | `code=422` | `NTW_ERR` | ✓ |

---

### 4-5. 일일 기업 과금 배치

**대상**: `DailyCompanyPayBatchService`

**▶ 테스트 플로우**

**1. 배치 수동 실행** — `GET /api/usage/b2b/manual-batch/dailyCompanyPayBatch`

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success"
}
```

`tb_pay_payment` 저장 데이터:
```
payment_id    : e52a091b-e70c-42ef-bbcb-f61e7ec578b1
subscribe_user_id : a598e6b2-4392-11f1-963e-7a06ad5292e5
admin_card_id : ddc58e38-4396-11f1-963e-7a06ad5292e5
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260626
pay_auth_code : 00539229
tid           : INIAPICARDINIBillTst20260626165629371235
price         : 2200
regdtm        : 2026-06-26 16:57:03
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 200,
  "msg": "success"
}
```

> 배치 자체는 정상 완료. 개별 결제 실패는 `tb_pay_payment.result_code`로 기록됨.

`tb_pay_payment` 저장 데이터:
```
payment_id    : 9a41f326-038c-4796-bcaa-15e9d9886db1
subscribe_user_id : a598e6b2-4392-11f1-963e-7a06ad5292e5
admin_card_id : ddc58e38-4396-11f1-963e-7a06ad5292e5
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 01
result_msg    : [1113][실시간빌링실패|요청금액 0원 이하]
pay_date      : 20260626
pay_auth_code : ERROR
tid           : INIAPICARDINIBillTst20260626170424931176
price         : 0
regdtm        : 2026-06-26 17:04:58
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 200,
  "msg": "success"
}
```

> 배치 자체는 정상 완료. 개별 결제 통신 오류는 `tb_pay_payment.result_code = 'NTW_ERR'`로 기록됨.

`tb_pay_payment` 저장 데이터:
```
payment_id    : a10be0c8-44e9-47d1-a1fa-dc9cb3754213
subscribe_user_id : a598e6b2-4392-11f1-963e-7a06ad5292e5
admin_card_id : ddc58e38-4396-11f1-963e-7a06ad5292e5
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-26 17:01:55
```

---

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| A. 결제 성공 | `code=200` | `00` | ✓ |
| B. 결제 실패 | `code=200` | `01` | ✓ |
| C. 통신 오류 | `code=200` | `NTW_ERR` | ✓ |

---

### 4-6. 개인사용자 과금 배치 (PgClientPaymentRequestBatch)

**대상**: `PgClientPaymentRequestBatchService`

**▶ 테스트 플로우**

**1. 배치 수동 실행** — `GET /api/usage/b2c/manual-batch/pgClientPaymentRequestBatch`

---

**▶ 테스트 결과**

**A. 결제 성공**

서버 응답:
```json
{
  "code": 200,
  "msg": "success"
}
```

> 배치 자체는 정상 완료. 개별 결제 성공은 `tb_pay_payment.result_code = '00'`으로 기록됨.

`tb_pay_payment` 저장 데이터:
```
payment_id    : 3ffd9728-0f82-4c7a-bf6f-d0fcf41a637b
subscribe_user_id : 0b9a9409-3aad-475a-be22-3b3db4c28067
admin_card_id : 4f5e5b12-96c3-485a-b608-c24fe89279df
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 00
result_msg    : [신용카드|빌링이 정상적으로 이루어졌습니다.]
pay_date      : 20260626
pay_auth_code : 00379124
tid           : INIAPICARDINIBillTst20260626171706232238
price         : 25300
regdtm        : 2026-06-26 17:18:22
```

---

**B. 결제 실패**

서버 응답:
```json
{
  "code": 200,
  "msg": "success"
}
```

> 배치 자체는 정상 완료. 개별 결제 실패는 `tb_pay_payment.result_code`로 기록됨.

`tb_pay_payment` 저장 데이터:
```
payment_id    : 463bd724-ae77-482f-8946-2f6c8ac67295
subscribe_user_id : 0b9a9409-3aad-475a-be22-3b3db4c28067
admin_card_id : 4f5e5b12-96c3-485a-b608-c24fe89279df
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : 01
result_msg    : [1113][실시간빌링실패|요청금액 0원 이하]
pay_date      : 20260626
pay_auth_code : ERROR
tid           : INIAPICARDINIBillTst20260626172826733107
price         : 0
regdtm        : 2026-06-26 17:28:47
```

---

**C. 통신 오류**

서버 응답:
```json
{
  "code": 200,
  "msg": "success"
}
```

> 배치 자체는 정상 완료. 개별 결제 통신 오류는 `tb_pay_payment.result_code = 'NTW_ERR'`로 기록됨.

`tb_pay_payment` 저장 데이터:
```
payment_id    : 1c317a16-7f0a-4b95-bbf2-d09fca340be3
subscribe_user_id : 0b9a9409-3aad-475a-be22-3b3db4c28067
admin_card_id : 4f5e5b12-96c3-485a-b608-c24fe89279df
bill_key      : 7971dbf8bb448efeb0d6eff1c88676337b8755a9
result_code   : NTW_ERR
result_msg    : 통신오류
regdtm        : 2026-06-26 17:23:13
```

---

| 케이스 | 서버 응답 | `result_code` | 통과 여부 |
|--------|---------|--------------|---------|
| A. 결제 성공 | `code=200` | `00` | ✓ |
| B. 결제 실패 | `code=200` | `01` | ✓ |
| C. 통신 오류 | `code=200` | `NTW_ERR` | ✓ |
