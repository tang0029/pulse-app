**Initiative Name**: Blood Pressure Tracker App for Elderly Adults  
**PRD Owner**: Tiara Anggamulia
**Date Created**: July 2026
**Last Updated**: September 2026
**Status**: Draft  
**Version**: 1.0

---

## 1. Problem

### What problem is this solving?
Elderly adults with hypertension need a simple, reliable way to track their daily blood pressure readings and share this information with family members and healthcare providers. Current solutions are either too complex (overwhelming comprehensive platforms), lack family sharing features, or require expensive hardware investments.

### Why is this problem worth solving?
Current blood pressure tracking apps are either too complex for elderly users or lack critical features like family monitoring and medication reminders. There's a significant gap in the market for a solution that combines simplicity, accessibility, and comprehensive health management specifically designed for elderly adults and their family members.


## 2. User

### Who are you solving this problem for?
Primary users are elderly adults (65+) managing hypertension, with secondary users being family members who need to monitor their loved one's health data.

### Current state
- Users track blood pressure manually in notes/spreadsheets (as evidenced by the data file)
- Existing apps are either too complex or lack critical features (family sharing, medication reminders)
- Device-specific apps require hardware purchases
- Frequent upgrade prompts and unclear free/premium differentiation create frustration

### Impact
- Inconsistent tracking leads to incomplete health data
- Family members cannot easily monitor elderly relatives' health
- Healthcare providers receive incomplete or poorly formatted data
- Medication adherence is not tracked alongside blood pressure readings
- Risk of health complications due to poor monitoring


## 3. Goals, Core Metrics & Prioritisation

### Primary goals
- Enable consistent daily blood pressure tracking (AM/PM readings) with minimal cognitive load
- Provide family members with real-time access to health data and alerts
- Generate clear, shareable reports for healthcare providers
- Support medication tracking and reminders to improve adherence

### How will you know the problem is solved? (Core Metrics)
| Metric                            | Baseline             | Target         | Measurement Method                                   |
| --------------------------------- | -------------------- | -------------- | ---------------------------------------------------- |
| Daily active usage rate           | 0% (manual tracking) | 90%+           | % of days with at least one reading logged           |
| Family/caregiver adoption         | 0%                   | 100%           | Check on Day 1  |
| Medication reminder adherence     | N/A                  | 85%+           | % of medication reminders acknowledged (Day 2)               |
| Healthcare provider sharing       | 0%                   | 100%          | Either the GP visit happens or not      |

### Prioritisation

#### Prioritised Stories (MVP Scope)
1. **As an** elderly adult, **I want** to quickly log my daily AM and PM blood pressure readings (systolic, diastolic, pulse), **so that** I can track my health without getting overwhelmed by complex features.
   - Acceptance criteria:
     - Large, easy-to-read input fields
     - One-tap time selection (AM/PM)
     - Voice input option for numbers
     - Confirmation screen before saving
     - Success feedback after saving

2. **As an** elderly adult, **I want** to see clear visual charts of my blood pressure trends over time, **so that** I can understand if my readings are improving or concerning.
   - Acceptance criteria:
     - Large, high-contrast charts
     - Colour-coded zones (normal, elevated, high)
     - Simple line graphs showing systolic, diastolic, and pulse trends
     - Ability to view by day, week, month
     - Clear labels and legends

3. **As a** family member, **I want** to view my elderly parent's blood pressure data in real-time, **so that** I can monitor their health remotely and intervene if needed.
   - Acceptance criteria:
     - Separate family member dashboard/app access
     - Real-time sync of blood pressure readings
     - View trends and current readings
     - Alert notifications for concerning readings
     - Secure authentication and privacy controls

4. **As an** elderly adult, **I want** to share my blood pressure data with my doctor/GP in a clear, professional format, **so that** they can make informed decisions about my treatment.
   - Acceptance criteria:
     - One-tap report generation
     - PDF export with charts and summary statistics
     - Email sharing capability
     - Date range selection
     - Clear, medical-grade formatting


## 4. Solution Overview

### High Level Approach
The solution is a web-based app designed with elderly users as the primary persona from the ground up. The app prioritises simplicity, accessibility, and family connectivity over feature complexity. It supports manual entry, ensuring broad accessibility regardless of device ownership.

### Key Features & Capabilities

#### Feature 1: Elderly-First User Interface
**Description**: Interface designed specifically for elderly users with large fonts (minimum 18pt), high contrast (WCAG AA compliance), simplified navigation (max 3 taps to any feature), clear visual hierarchy, and minimal cognitive load.

**User value**: Reduces frustration, increases adoption, and enables independent use by elderly adults with varying tech proficiency.

**Technical considerations**: 
- Responsive design supporting tablet and phone
- Dynamic font scaling respecting system settings
- Colour-blind friendly palette
- Touch targets minimum 44x44pt

#### Feature 2: Daily Blood Pressure Tracking (AM/PM)
**Description**: Simple, focused interface for logging systolic, diastolic, and pulse readings with AM/PM time indicators. Supports voice input and manual entry.

**User value**: Matches real-world usage patterns (daily AM/PM readings) and reduces data entry errors.

**Technical considerations**:
- Voice-to-text integration for number input
- Input validation (reasonable ranges: systolic 80-250, diastolic 40-150, pulse 40-120)
- Quick entry mode for frequent users
- Data persistence with cloud backup

#### Feature 3: Family Dashboard
**Description**: Separate family member interface (web app) providing real-time view of elderly user's blood pressure data, trend analysis, alert configuration, and communication tools.

**User value**: Enables remote health monitoring by family members, addresses critical gap in market.

**Technical considerations**:
- Secure authentication and authorisation
- Real-time data sync
- Privacy controls (elderly user and family member can grant/revoke access)
- Push notification system for alerts

#### Feature 4: Trend Visualisation
**Description**: Clear, large-format charts showing systolic, diastolic, and pulse trends over time with colour-coded zones indicating normal/elevated/high ranges.

**User value**: Helps users understand their health patterns and identify concerning trends.

**Technical considerations**:
- Charting library (responsive, accessible)
- Colour-coding based on AHA guidelines
- Multiple time views (day, week, month, year)
- Export capabilities

#### Feature 5: Healthcare Provider Reports
**Description**: One-tap report generation with professional formatting, charts, summary statistics, and multiple sharing options (email, PDF, print).

**User value**: Facilitates communication with healthcare providers and improves care coordination.

**Technical considerations**:
- PDF generation library
- Chart rendering for reports
- Email integration
- Date range selection and filtering


## 5. Dependencies, Constraints & Assumptions

### Dependencies
#### Internal
  - Design system and UI components
  - Backend infrastructure setup
  - CI/CD pipeline for cloud-based deployments

#### External
  - Compliance for GDPR and New Zealand Privacy Act 2020
  - Accessibility testing with elderly users

### Constraints
#### Regulatory
  - Medical device regulations (not a medical device, but health data)
  - Privacy regulations (GDPR, New Zealand Privacy Act 2020)

### Assumptions
- Internet connectivity available for initial setup and sync 
- Users are comfortable with basic app navigation after onboarding
- Family members have computers/tablets for dashboard access


## 6. Technical Requirements

### Architecture Considerations
**Platform**: Web app for optimal performance and accessibility
- **Frontend:** Cloud-based modern UI components, e.g. React, TypeScript
- **Backend**: Cloud-based for data sync and family sharing
- **Database**: Secure, encrypted storage with GDPR and New Zealand Privacy Act 2020 considerations, e.g. Supabase Postgres database
- **API**: RESTful APIs 
- **Hosting:** Cloud-based and AI-based hosting platform, e.g. Netlify

### Data Requirements
#### Data Sources
  - User-entered BP readings (systolic, diastolic, pulse, timestamp, AM/PM)
  - Medication data (name, dosage, frequency, reminder times)
  - User notes and tags
  - Family members and permissions

#### Data Storage
  - Encrypted at rest and in transit
  - Cloud database for sync and sharing
  - Backup and recovery procedures

#### Data Accuracy
  - Input validation (range checks)
  - Device reading validation
  - Timestamp accuracy (timezone handling)
  - Data integrity checks

### Performance Requirements
- Web load time: <2 seconds on desktops, tablets and mobile devices
- Reading entry: <1 second to save
- Chart rendering: <1 second for 30 days of data
- Sync: Automatic background sync

### Security & Compliance
**Authentication**: Secure login with email address
- **Data encryption**: End-to-end encryption for sensitive health data
- **Privacy**: 
  - User controls over data sharing
  - Clear privacy policy
  - GDPR and New Zealand Privacy Act 2020 considerations 
  - No data sharing with third parties without explicit consent
- **Access controls**: Role-based access (elderly users, family members)
- **Audit logging**: Track data access and modifications

## 7. Risks & Mitigation

### High Risk
**Risk**: Elderly users find app too complex despite design efforts
  - **Impact**: Low adoption, user frustration, product failure
  - **Probability**: Medium
  - **Mitigation**: Extensive user testing with elderly users throughout design and development, iterative improvements based on feedback, simplified onboarding flow

- **Risk**: Privacy/regulatory compliance issues (GDPR, New Zealand Privacy Act 2020)
  - **Impact**: Legal issues, user trust concerns
  - **Probability**: Medium
  - **Mitigation**: Privacy-by-design approach, clear privacy policy, user consent flows

## 8. Production Readiness Criteria & Metrics

### Prod Readiness Criteria

### Metrics

