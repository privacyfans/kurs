# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Laravel 11 Exchange Rate Management System** with markup capabilities and approval workflows. The system integrates with Oracle database for real-time exchange rate data and provides both web interface and REST API access.

## Key Architecture Components

### Technology Stack
- **Backend**: Laravel 11 (PHP 8.4.10)
- **Web Server**: Apache 2.4
- **Database**: Oracle (external data) + MySQL/PostgreSQL (application data)
- **Authentication**: JWT (JSON Web Token)
- **Cache**: Redis for performance optimization
- **Queue**: Laravel Queue for background jobs

### Core System Features
1. **Automated Exchange Rate Updates**: Scheduled every hour via cron job
2. **Role-Based Access Control**: Maker, Checker, Viewer, and Admin roles
3. **Markup Management Workflow**: Approval-based markup application system
4. **REST API**: JWT-authenticated endpoints with rate limiting
5. **Audit Trail**: Complete history tracking for compliance

## Database Schema

### Main Tables
- `users` - User management with roles
- `exchange_rates_raw` - Raw data from Oracle
- `exchange_rates` - Final rates with markup applied
- `markup_proposals` - Markup approval workflow
- `api_tokens` - API access token management

## Oracle Integration

### Data Source Query
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

## API Endpoints

Base URL: `/api/v1/exchange-rates`
Authentication: JWT Bearer Token
Rate Limiting: 100 requests/minute per token

### Available Endpoints
1. `GET /api/v1/exchange-rates` - All current rates
2. `GET /api/v1/exchange-rates?currency={FRM_CUR_CD}` - Rates by currency
3. `GET /api/v1/exchange-rates?date={NTC_DT}` - Rates by date (YYYYMMDD or YYYY-MM-DD)
4. `GET /api/v1/exchange-rates?currency={FRM_CUR_CD}&date={NTC_DT}` - Combined filter

### Response Format
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

## Business Rules

### Markup Workflow
- Only one pending markup per currency allowed
- Approved markups apply to next rate update
- Rejected markups can be resubmitted with different values
- All markup history preserved for audit

### Rate Calculation
- Final Rate = Raw Rate + (Raw Rate × Markup Percentage / 100)
- All rates rounded to 4 decimal places

### Role Permissions
- **Maker**: Input markup, view own history, cancel pending submissions
- **Checker**: Approve/reject markups, bulk approval capability
- **Viewer**: View rates, download historical data, generate API tokens
- **Admin**: Full system access

## Performance Requirements
- Page load time < 3 seconds
- API response time < 2 seconds
- Support for 100+ concurrent users
- 99.9% system uptime target

## Security Features
- JWT authentication with 24-hour expiry
- Session timeout: 30 minutes idle
- SSL/TLS encryption required
- Input validation and SQL injection prevention
- XSS protection implemented

Note: This project is currently in early planning/BRD phase. The actual Laravel application structure and development commands will be added once implementation begins.