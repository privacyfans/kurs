# Business Requirements Document (BRD)
## Sistem Informasi Kurs dengan Markup Management

---

## 1. EXECUTIVE SUMMARY

### 1.1 Latar Belakang
Perusahaan memerlukan sistem informasi kurs yang terintegrasi dengan database Oracle untuk menampilkan informasi kurs terkini dengan kemampuan menambahkan persentase markup. Sistem ini harus dapat diakses melalui aplikasi web dan API dengan kontrol akses berbasis role.

### 1.2 Tujuan Proyek
- Menyediakan informasi kurs real-time yang diperbarui setiap jam dari database Oracle
- Memungkinkan pengelolaan markup kurs dengan workflow approval
- Menyediakan API untuk konsumsi data kurs oleh sistem eksternal
- Mengimplementasikan kontrol akses berbasis role untuk keamanan data

### 1.3 Ruang Lingkup
Pengembangan aplikasi web dan API untuk manajemen informasi kurs dengan fitur markup, approval workflow, dan historical tracking.

---

## 2. BUSINESS OBJECTIVES

### 2.1 Objektif Utama
1. **Otomasi Update Kurs**: Mengotomasi pengambilan data kurs dari Oracle setiap 1 jam
2. **Manajemen Markup**: Menyediakan sistem markup kurs dengan approval workflow
3. **Aksesibilitas Data**: Menyediakan akses data melalui web interface dan REST API
4. **Audit Trail**: Mencatat semua perubahan markup dan approval untuk keperluan audit

### 2.2 Key Performance Indicators (KPI)
- Update kurs tepat waktu (setiap 1 jam)
- Response time API < 2 detik
- Uptime sistem 99.9%
- Zero data breach incidents

---

## 3. STAKEHOLDER ANALYSIS

### 3.1 Internal Stakeholders
| Stakeholder | Role | Kepentingan |
|------------|------|-------------|
| Finance Team | Maker | Input dan manage markup kurs |
| Finance Manager | Checker | Approve/reject markup proposal |
| Business Units | Viewer | Konsumsi data kurs untuk operasional |
| IT Department | Admin | Maintenance dan monitoring sistem |

### 3.2 External Stakeholders
- Partner systems yang mengkonsumsi API
- Auditor untuk compliance checking

---

## 4. FUNCTIONAL REQUIREMENTS

### 4.1 Role Management

#### 4.1.1 Maker Role
- **Akses Menu**: Dashboard, Input Markup, History Markup (own data)
- **Fungsi Utama**:
  - View current exchange rates
  - Input persentase markup untuk setiap currency
  - Edit markup yang belum di-approve
  - View status approval markup yang diajukan
  - Cancel markup submission sebelum di-review

#### 4.1.2 Checker Role
- **Akses Menu**: Dashboard, Approval Queue, History Approval
- **Fungsi Utama**:
  - View pending markup proposals
  - Approve markup dengan remarks
  - Reject markup dengan alasan
  - View history approval yang telah dilakukan
  - Bulk approval untuk multiple currencies

#### 4.1.3 Viewer Role
- **Akses Menu**: Dashboard, History Kurs, API Documentation
- **Fungsi Utama**:
  - View current exchange rates dengan markup
  - View historical exchange rates
  - Generate API token untuk akses programmatic
  - Download historical data dalam format Excel/CSV

### 4.2 Core Features

#### 4.2.1 Automated Exchange Rate Update
- **Scheduler**: Cron job setiap 1 jam
- **Process Flow**:
  1. Execute Oracle query pada menit ke-00 setiap jam
  2. Parse hasil query
  3. Store raw data ke tabel `exchange_rates_raw`
  4. Calculate final rate dengan approved markup
  5. Store final data ke tabel `exchange_rates`
  6. Update cache untuk quick access

#### 4.2.2 Markup Management Workflow
```
[Maker Input] → [Pending] → [Checker Review] → [Approved/Rejected]
                    ↓                                    ↓
                [Cancel]                          [Applied to Rates]
```

**Business Rules**:
- Satu currency hanya boleh memiliki 1 pending markup
- Markup yang di-approve langsung berlaku pada update kurs berikutnya
- Markup yang di-reject dapat diajukan kembali dengan nilai berbeda
- History markup tersimpan permanent untuk audit

#### 4.2.3 API Service
- **Base Endpoint**: `/api/v1/exchange-rates`
- **Authentication**: JWT Bearer Token
- **Rate Limiting**: 100 requests per minute per token

**Available Endpoints**:

1. **Get All Current Rates**
   - **Method**: GET
   - **URL**: `/api/v1/exchange-rates`
   - **Description**: Retrieve all current exchange rates

2. **Get Rates by Currency**
   - **Method**: GET
   - **URL**: `/api/v1/exchange-rates?currency={FRM_CUR_CD}`
   - **Parameters**: 
     - `currency` (optional): Currency code (e.g., USD, EUR, KRW)
   - **Example**: `/api/v1/exchange-rates?currency=USD`

3. **Get Rates by Date**
   - **Method**: GET
   - **URL**: `/api/v1/exchange-rates?date={NTC_DT}`
   - **Parameters**:
     - `date` (optional): Date in format YYYYMMDD or YYYY-MM-DD
   - **Example**: `/api/v1/exchange-rates?date=20250605`

4. **Get Rates by Currency and Date**
   - **Method**: GET
   - **URL**: `/api/v1/exchange-rates?currency={FRM_CUR_CD}&date={NTC_DT}`
   - **Parameters**:
     - `currency` (optional): Currency code
     - `date` (optional): Date in format YYYYMMDD or YYYY-MM-DD
   - **Example**: `/api/v1/exchange-rates?currency=USD&date=20250605`

**Response Format**:
```json
{
    "rc": "00",
    "msg": "Success",
    "data": [
        {
            "update_date": "2025-08-06 08:32:12",
            "currency": "KRW",
            "kurs": "11.9900"
        }
    ],
    "response_date": "2025-08-06 17:32:12"
}
```

**Error Response Codes**:
- `00`: Success
- `01`: Unauthorized access
- `02`: Invalid parameters
- `03`: Data not found
- `04`: Rate limit exceeded
- `99`: System error

### 4.3 Data Management

#### 4.3.1 Oracle Query Integration
```sql
select * from (
   select NTC_DT ,FRM_CUR_CD,NTC_XRT_KD_DSCD, sum(NTC_XRT) NTC_XRT
   from ACOWN.GBC21003TF 
   where NTC_DT = '20250605' 
   AND NTC_SQ = (select max(NTC_SQ) from ACOWN.GBC21003TF where NTC_SQ<>'9999' and NTC_DT = '20250605') 
   AND TO_CUR_CD = 'IDR'
   AND NTC_XRT_KD_DSCD = 'TTS'
   group by BK_CD, NTC_DT, NTC_SQ, FRM_CUR_CD, TO_CUR_CD, NTC_XRT_KD_DSCD 
)
pivot 
(
   sum(NTC_XRT)
   for NTC_XRT_KD_DSCD in ('TTS' as TTS)
)
```

#### 4.3.2 Data Processing
- **Raw Rate**: Nilai TTS dari Oracle query
- **Markup Calculation**: `Final Rate = Raw Rate + (Raw Rate × Markup Percentage / 100)`
- **Rounding**: 4 decimal places untuk semua currency (contoh: 11.9900, 16370.0000)

---

## 5. NON-FUNCTIONAL REQUIREMENTS

### 5.1 Performance Requirements
- Page load time < 3 seconds
- API response time < 2 seconds
- Concurrent users support: minimum 100 users
- Database query optimization untuk large datasets

### 5.2 Security Requirements
- JWT authentication dengan expiry time 24 hours
- Password policy: minimum 8 characters, alphanumeric with special characters
- Session timeout: 30 minutes idle time
- SSL/TLS encryption untuk semua communications
- Input validation dan sanitization
- SQL injection prevention
- XSS protection

### 5.3 Availability Requirements
- System availability: 99.9% uptime
- Scheduled maintenance window: Sundays 00:00-04:00 WIB
- Backup frequency: Daily incremental, Weekly full backup
- Disaster recovery: RPO 1 hour, RTO 4 hours

### 5.4 Compatibility Requirements
- Browser support: Chrome 90+, Firefox 88+, Safari 14+, Edge 90+
- Mobile responsive design
- API backward compatibility untuk 2 major versions

---

## 6. TECHNICAL ARCHITECTURE

### 6.1 Technology Stack
- **Backend Framework**: Laravel 11
- **Programming Language**: PHP 8.4.10
- **Web Server**: Apache 2.4
- **Database**: Oracle (existing) + MySQL/PostgreSQL (application data)
- **Authentication**: JWT (JSON Web Token)
- **Cache**: Redis untuk performance optimization
- **Queue**: Laravel Queue untuk background jobs

### 6.2 System Architecture
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Web UI    │────▶│   Laravel   │────▶│   Oracle    │
└─────────────┘     │  Application│     │   Database  │
                    └─────────────┘     └─────────────┘
┌─────────────┐            │            ┌─────────────┐
│   API       │────────────┼───────────▶│   Redis     │
│   Consumer  │            │            │   Cache     │
└─────────────┘     ┌─────────────┐    └─────────────┘
                    │   Scheduler  │
                    │   (Cron)     │
                    └─────────────┘
```

### 6.3 Database Schema

#### Tables Structure
1. **users**
   - id, email, password, role, status, created_at, updated_at

2. **exchange_rates_raw**
   - id, currency, rate, fetch_date, created_at

3. **exchange_rates**
   - id, currency, raw_rate, markup_percentage, final_rate, effective_date, created_at

4. **markup_proposals**
   - id, currency, proposed_markup, maker_id, checker_id, status, remarks, created_at, updated_at

5. **api_tokens**
   - id, user_id, token, expires_at, last_used_at, created_at

---

## 7. USER INTERFACE REQUIREMENTS

### 7.1 Dashboard
- **Real-time Exchange Rate Display**: Grid/table format dengan auto-refresh
- **Quick Stats**: Total currencies, last update time, pending approvals
- **Charts**: Historical rate trends (optional enhancement)

### 7.2 Markup Management Interface
- **Input Form**: Dropdown currency selection, percentage input, remarks field
- **Validation**: Real-time validation dengan preview final rate
- **Bulk Input**: Option untuk input multiple currencies sekaligus

### 7.3 Approval Interface
- **Queue Display**: Sortable/filterable table
- **Detail View**: Modal/sidebar dengan complete information
- **Action Buttons**: Approve/Reject dengan mandatory remarks

### 7.4 History & Reporting
- **Filter Options**: Date range, currency, status
- **Export Functions**: Excel, CSV, PDF formats
- **Pagination**: Server-side pagination untuk large datasets

---

## 8. IMPLEMENTATION PLAN

### 8.1 Development Phases

#### Phase 1: Foundation (2 weeks)
- Environment setup
- Database design dan migration
- Authentication system (JWT)
- Basic role management

#### Phase 2: Core Features (3 weeks)
- Oracle integration
- Automated scheduler
- Markup management workflow
- Basic UI implementation

#### Phase 3: API Development (1 week)
- REST API endpoints
- JWT token management
- API documentation
- Rate limiting implementation

#### Phase 4: Testing & Refinement (2 weeks)
- Unit testing
- Integration testing
- UAT (User Acceptance Testing)
- Performance optimization

#### Phase 5: Deployment (1 week)
- Production environment setup
- Data migration
- Go-live preparation
- Post-deployment monitoring

### 8.2 Total Timeline
**Estimated Duration**: 9 weeks

---

## 9. TESTING REQUIREMENTS

### 9.1 Testing Scope
- **Unit Testing**: Minimum 80% code coverage
- **Integration Testing**: API endpoints, Oracle connectivity
- **Performance Testing**: Load testing untuk 100 concurrent users
- **Security Testing**: Penetration testing, vulnerability assessment
- **UAT**: Business scenario testing dengan actual users

### 9.2 Test Scenarios
1. Exchange rate update accuracy
2. Markup calculation correctness with 4 decimal places
3. Approval workflow integrity
4. API response format validation
5. API parameter filtering (currency and date)
6. Role-based access control
7. Data consistency during high load
8. Failover dan recovery procedures
9. Date format validation (YYYYMMDD dan YYYY-MM-DD)
10. Currency code validation

---

## 10. RISK ASSESSMENT

### 10.1 Technical Risks
| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Oracle DB connectivity issues | Medium | High | Implement connection pooling dan retry mechanism |
| Performance degradation | Low | Medium | Caching strategy dan query optimization |
| Security vulnerabilities | Low | High | Regular security audits dan updates |

### 10.2 Business Risks
| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Incorrect markup application | Low | High | Dual approval mechanism dan audit trails |
| Data inconsistency | Low | High | Transaction management dan validation rules |
| User adoption challenges | Medium | Medium | Comprehensive training dan documentation |

---

## 11. SUCCESS CRITERIA

### 11.1 Acceptance Criteria
- ✓ All roles can perform designated functions without errors
- ✓ Exchange rates update automatically every hour
- ✓ Markup workflow completes end-to-end successfully
- ✓ API returns data in specified format
- ✓ System handles 100 concurrent users
- ✓ All security requirements are met

### 11.2 Post-Implementation Review (After 3 months)
- User satisfaction survey
- System performance metrics review
- Incident dan issue analysis
- Process improvement recommendations

---

## 12. MAINTENANCE & SUPPORT

### 12.1 Maintenance Plan
- **Regular Updates**: Security patches monthly
- **Feature Enhancements**: Quarterly review
- **Database Maintenance**: Weekly optimization
- **Backup Verification**: Monthly restore testing

### 12.2 Support Structure
- **L1 Support**: Help desk untuk basic issues
- **L2 Support**: Application team untuk functional issues
- **L3 Support**: Development team untuk critical bugs

---

## 13. APPENDICES

### Appendix A: Glossary
- **TTS**: Telegraphic Transfer Selling rate
- **Markup**: Additional percentage added to base exchange rate
- **JWT**: JSON Web Token untuk secure authentication
- **Cron**: Time-based job scheduler

### Appendix B: Sample API Responses

**Success Response - All Currencies**:
```json
{
    "rc": "00",
    "msg": "Success",
    "data": [
        {
            "update_date": "2025-08-06 08:32:12",
            "currency": "USD",
            "kurs": "16534.7000"
        },
        {
            "update_date": "2025-08-06 08:32:12",
            "currency": "EUR",
            "kurs": "18914.4619"
        },
        {
            "update_date": "2025-08-06 08:32:12",
            "currency": "KRW",
            "kurs": "11.9900"
        }
    ],
    "response_date": "2025-08-06 17:32:12"
}
```

**Success Response - Filtered by Currency**:
```json
{
    "rc": "00",
    "msg": "Success",
    "data": [
        {
            "update_date": "2025-08-06 08:32:12",
            "currency": "USD",
            "kurs": "16534.7000"
        }
    ],
    "response_date": "2025-08-06 17:32:12"
}
```

**Success Response - Filtered by Date**:
```json
{
    "rc": "00",
    "msg": "Success",
    "data": [
        {
            "update_date": "2025-06-05 09:00:00",
            "currency": "USD",
            "kurs": "16370.0000"
        },
        {
            "update_date": "2025-06-05 09:00:00",
            "currency": "KRW",
            "kurs": "11.9900"
        }
    ],
    "response_date": "2025-08-06 17:32:12"
}
```

**Error Response - Invalid Parameters**:
```json
{
    "rc": "02",
    "msg": "Invalid date format. Please use YYYYMMDD or YYYY-MM-DD",
    "data": [],
    "response_date": "2025-08-06 17:32:12"
}
```

**Error Response - Data Not Found**:
```json
{
    "rc": "03",
    "msg": "No exchange rate data found for the specified criteria",
    "data": [],
    "response_date": "2025-08-06 17:32:12"
}
```

**Error Response - Unauthorized**:
```json
{
    "rc": "01",
    "msg": "Unauthorized access. Invalid or expired token",
    "data": [],
    "response_date": "2025-08-06 17:32:12"
}
```

### Appendix C: Role Permission Matrix

| Feature | Maker | Checker | Viewer | Admin |
|---------|-------|---------|--------|-------|
| View Dashboard | ✓ | ✓ | ✓ | ✓ |
| Input Markup | ✓ | - | - | ✓ |
| Approve Markup | - | ✓ | - | ✓ |
| View History | ✓ | ✓ | ✓ | ✓ |
| API Access | - | - | ✓ | ✓ |
| User Management | - | - | - | ✓ |

---

**Document Version**: 1.0  
**Date**: August 2025  
**Status**: Draft  
**Author**: System Analyst Team  
**Approval**: Pending