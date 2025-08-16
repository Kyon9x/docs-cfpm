# CFPM Timezone Management Enhancement PRD

## Intro Project Analysis and Context

### Analysis Source
- IDE-based fresh analysis of existing CFPM Frontend and Backend codebase
- Review of current user entity structure, date handling utilities, and UI components
- Assessment of timezone impact across transaction displays, calendars, and user profiles

### Current Project State

**Project Purpose**: CFPM (Comprehensive Financial Portfolio Management) is a full-stack financial portfolio management application that enables users to track wallets, portfolios, assets, and transactions with real-time market data integration.

**Current Timezone Handling**: 
- All dates are stored in UTC in the PostgreSQL database
- Frontend displays dates using `formatDateDDMMYYYY()` utility without timezone conversion
- Users experience timezone confusion, particularly for GMT+7 users seeing UTC timestamps
- Time displays use `toLocaleTimeString('vi-VN')` but still show UTC-based times

### Available Documentation Analysis

**Available Documentation**:
- ✓ Tech Stack Documentation (NestJS backend, React TypeScript frontend)
- ✓ Source Tree/Architecture (modular NestJS structure, component-based React)  
- ✓ API Documentation (Swagger-documented endpoints)
- ✓ External API Documentation (Price service integration)
- ✓ Technical Debt Documentation (Known UTC timezone display issues)

**Missing Documentation**: UX/UI Guidelines for timezone preferences

### Enhancement Scope Definition

**Enhancement Type**: ☑ New Feature Addition

**Enhancement Description**: Add comprehensive timezone support to allow users to set their preferred timezone and display all dates/times in their local timezone while maintaining UTC storage in the database.

**Impact Assessment**: ☑ Moderate Impact (some existing code changes) - Will require updates to date utilities, user entity, profile UI, and all date display components, but maintains existing API contracts.

### Goals and Background Context

**Goals**:
- Enable users to set their preferred timezone in profile settings
- Display all dates and times in the user's selected timezone instead of UTC
- Maintain data integrity by continuing to store all timestamps in UTC
- Provide seamless timezone experience for users in different geographical locations
- Support common Asian timezones with focus on GMT+7 (Vietnam/Thailand)

**Background Context**: 

Currently, users in GMT+7 (Vietnam, Thailand) see transaction times and dates displayed in UTC, causing confusion when tracking their financial activities. For example, a transaction made at 2:00 PM local time appears as 7:00 AM UTC, making it difficult to correlate with actual trading activities. This enhancement will solve the user experience problem while maintaining technical best practices of UTC storage.

The existing user entity already supports `locale` and `currency` preferences, establishing a pattern for user-specific display preferences that this enhancement will extend.

### Change Log

| Change | Date | Version | Description | Author |
|--------|------|---------|-------------|---------|
| Initial PRD Creation | 2025-01-16 | 1.0 | Created comprehensive timezone management PRD | AI Assistant |

## Requirements

Based on analysis of the existing CFPM system with NestJS backend, React TypeScript frontend, PostgreSQL database, and current UTC-only date handling, the following requirements ensure seamless timezone integration while maintaining system integrity.

### Functional

- **FR1**: The user entity shall include a `timezone` field supporting IANA timezone database format (e.g., 'Asia/Ho_Chi_Minh', 'America/New_York') with default value 'UTC' for backward compatibility
- **FR2**: The user profile settings page shall provide a timezone selector dropdown integrated with existing locale and currency preferences using the same UI patterns
- **FR3**: All date and time displays throughout the application (TransactionTable, TransactionPage calendar, RecentActivityCard, etc.) shall convert UTC timestamps to the user's selected timezone
- **FR4**: The system shall maintain backward compatibility for existing users without timezone preference, defaulting to UTC display until they set a preference
- **FR5**: All database storage shall continue to use UTC timestamps without modification to existing schema or data
- **FR6**: The system shall handle daylight saving time transitions automatically based on the selected timezone
- **FR7**: Date formatting shall maintain existing locale preferences (DD/MM/YYYY for Vietnamese) while applying timezone conversion

### Non Functional

- **NFR1**: Timezone conversion operations shall not add more than 100ms latency to page load times or API responses
- **NFR2**: The system shall support all standard IANA timezone database entries with graceful fallback to UTC for invalid timezone strings
- **NFR3**: Timezone changes shall take effect immediately upon saving without requiring user session restart
- **NFR4**: The enhancement shall maintain existing performance characteristics for date-heavy pages like TransactionPage
- **NFR5**: Browser timezone detection may be used as an initial suggestion but shall not override explicit user selection
- **NFR6**: The system shall handle timezone data updates and maintain compatibility with timezone database changes

### Compatibility Requirements

- **CR1**: All existing API endpoints must continue to accept and return UTC timestamps in ISO 8601 format without breaking changes to external integrations
- **CR2**: Database schema continues to store all datetime columns in UTC, ensuring data consistency and simplifying backend operations
- **CR3**: Frontend components must handle both timezone-aware and legacy UTC-only API responses during transition period
- **CR4**: Existing date utility functions (`formatDateDDMMYYYY`, `formatDateYYYYMMDD`) must remain functional for components not yet updated with timezone support

## User Interface Enhancement Goals

### Integration with Existing UI

The timezone selector will integrate seamlessly with the existing ProfilePage.tsx design patterns:
- Use the same `Select`, `SelectContent`, `SelectItem` components already used for currency and locale selection  
- Follow the established form layout with `Label` and description text
- Maintain consistent spacing and visual hierarchy with existing preference controls
- Include edit/save functionality matching the current profile editing workflow

### Modified/New Screens and Views

**Modified Screens**:
- **ProfilePage.tsx**: Add timezone selector in the user preferences section
- **TransactionTable.tsx**: Display dates in user timezone with optional timezone indicator
- **TransactionPage.tsx**: Convert calendar dates and transaction timestamps
- **RecentActivityCard.tsx**: Show activity times in user timezone
- **WalletCard.tsx** and related wallet components: Convert timestamp displays

**New UI Elements**:
- Timezone selector dropdown with search/filter capability for 400+ timezones
- Optional timezone abbreviation display (e.g., "GMT+7") next to dates when space permits

### UI Consistency Requirements

- All timezone-converted dates must maintain the existing DD/MM/YYYY format for Vietnamese locale
- Time displays should continue to use 24-hour format as currently implemented
- Loading states and error handling for timezone operations should match existing patterns
- Timezone preference changes should provide immediate visual feedback similar to other profile updates

## Technical Constraints and Integration Requirements

### Existing Technology Stack

**Languages**: TypeScript (Frontend), TypeScript (Backend)
**Frameworks**: React 18+ with hooks (Frontend), NestJS 11.x (Backend)
**Database**: PostgreSQL with TypeORM 0.3.x
**Infrastructure**: Docker containerization, RESTful API architecture
**External Dependencies**: Swagger documentation, class-validator, JWT authentication

### Integration Approach

**Database Integration Strategy**: 
- Add `timezone` column to existing `users` table via TypeORM migration
- Default value 'UTC' for all existing users maintains compatibility
- No changes to transaction, wallet, or other timestamp storage formats
- Migration includes rollback capability for safe deployment

**API Integration Strategy**:
- Extend existing user DTOs (`CreateUserDto`, `UpdateUserDto`, `UserResponseDto`) 
- Add timezone validation using class-validator with IANA timezone list
- User profile endpoint `/users/profile` includes timezone in response
- No changes required to transaction or other timestamp-returning endpoints

**Frontend Integration Strategy**:
- Create timezone conversion utilities in `src/lib/utils/timezoneUtils.ts`
- Extend existing `dateUtils.ts` with timezone-aware formatting functions
- Implement timezone context provider for application-wide timezone access
- Update date display components incrementally without breaking existing functionality

**Testing Integration Strategy**:
- Unit tests for timezone conversion utilities with multiple timezone scenarios
- Integration tests for profile timezone update workflow
- E2E tests verifying timezone display consistency across major components
- Performance tests ensuring timezone operations don't exceed NFR1 latency requirements

### Code Organization and Standards

**File Structure Approach**:
- Backend: `/src/users/` directory contains timezone-related modifications
- Frontend: `/src/lib/utils/timezoneUtils.ts` for new timezone utilities
- Frontend: `/src/contexts/TimezoneContext.tsx` for application state management
- Migration: `/src/migrations/` directory for database schema update

**Naming Conventions**: 
- Follow existing camelCase for variables, PascalCase for components
- Timezone-related functions prefixed with `timezone` or `tz` (e.g., `timezoneFormatDate`)
- Use consistent naming pattern: `convertToUserTimezone`, `formatInUserTimezone`

**Coding Standards**:
- Maintain strict TypeScript typing, avoid `any` for timezone operations
- Follow existing NestJS decorator patterns for DTOs and validation
- Use React hooks patterns consistent with existing components
- Implement proper error handling with graceful fallback to UTC display

**Documentation Standards**:
- JSDoc comments for all new timezone utility functions
- Swagger documentation updates for modified DTOs
- README updates documenting timezone feature and configuration

### Deployment and Operations

**Build Process Integration**:
- No changes to existing build pipeline required
- Timezone data bundled with application, no external timezone service dependencies
- Frontend bundle size impact minimal (timezone utilities ~10KB)

**Deployment Strategy**:
- Database migration runs automatically on deployment
- Feature flag capability for gradual rollout (optional)
- Backward compatible deployment allows rollback without data loss

**Monitoring and Logging**:
- Log timezone conversion errors with fallback to UTC
- Monitor timezone preference adoption rates
- Track performance metrics for timezone operations

**Configuration Management**:
- Default timezone configuration via environment variables
- No additional external service configuration required

### Risk Assessment and Mitigation

**Technical Risks**:
- Risk: Inconsistent timezone handling across different browsers
- Mitigation: Use robust date-fns-tz library for reliable timezone operations
- Risk: Performance degradation with timezone calculations
- Mitigation: Implement timezone conversion caching and optimize calculation frequency

**Integration Risks**:
- Risk: Existing date comparisons may break with timezone conversion
- Mitigation: Keep all internal date operations in UTC, only convert for display
- Risk: Calendar component date selection issues with timezone boundaries
- Mitigation: Ensure date picker operations remain in UTC for consistency

**Deployment Risks**:
- Risk: Database migration failure could affect user authentication
- Mitigation: Test migration thoroughly in staging environment, include tested rollback script
- Risk: Frontend timezone errors causing blank date displays
- Mitigation: Implement comprehensive error boundaries with UTC fallback display

## Epic and Story Structure

**Epic Structure Decision**: Single comprehensive epic for brownfield timezone enhancement with integrated approach to maintain system consistency while delivering user value incrementally.

## Epic 1: User Timezone Management

**Epic Goal**: Enable users to set timezone preferences and display all dates/times in their selected timezone while maintaining UTC data storage and system integrity.

**Integration Requirements**: 
- Seamless integration with existing user preference patterns (locale/currency)
- Maintain all existing API contracts and database storage formats
- Ensure consistent timezone handling across all date-sensitive components
- Support existing Vietnamese localization and DD/MM/YYYY date formatting

### Story 1.1: Backend Timezone Support

As a system administrator,
I want to add timezone support to the user model and APIs,
so that user timezone preferences can be stored and retrieved reliably.

#### Acceptance Criteria

1. **Database Schema**: Add `timezone` column to `users` table with VARCHAR(50) type, default value 'UTC', and NOT NULL constraint
2. **Entity Update**: Update `User` entity class with timezone property including TypeORM column decorator
3. **DTO Validation**: Extend `CreateUserDto` and `UpdateUserDto` with timezone field and IANA timezone validation
4. **API Response**: Include timezone field in all user response DTOs (`UserResponseDto`, `CurrentUserResponseDto`)
5. **Swagger Documentation**: Update OpenAPI documentation with timezone field descriptions and examples
6. **Migration Safety**: Create reversible migration with proper rollback capability

#### Integration Verification

- **IV1**: Existing user authentication and profile retrieval continues to work without errors
- **IV2**: User registration process accepts timezone parameter without breaking existing flows  
- **IV3**: Database queries maintain existing performance characteristics with new column

### Story 1.2: Frontend Timezone Utilities

As a developer,
I want robust timezone conversion utilities,
so that date displays can be consistently converted from UTC to user's timezone.

#### Acceptance Criteria

1. **Timezone Utilities**: Create `timezoneUtils.ts` with functions for UTC to timezone conversion and IANA timezone validation
2. **Date Utils Extension**: Extend existing `dateUtils.ts` with timezone-aware versions of `formatDateDDMMYYYY` and `formatDateYYYYMMDD`
3. **Timezone Context**: Implement React context provider for application-wide timezone state management
4. **Error Handling**: Implement graceful fallback to UTC display for invalid timezone data
5. **Performance Optimization**: Cache timezone conversion results for repeated date formatting
6. **Type Safety**: Provide full TypeScript typing for all timezone operations

#### Integration Verification

- **IV1**: Existing date formatting functions continue to work unchanged for components not using timezone features
- **IV2**: New timezone utilities handle edge cases (DST transitions, invalid timezones) without crashing
- **IV3**: Context provider integrates with existing authentication context without conflicts

### Story 1.3: Profile Timezone Settings

As a user,
I want to set my timezone preference in my profile settings,
so that I can see dates and times displayed in my local timezone throughout the application.

#### Acceptance Criteria

1. **Timezone Selector**: Add timezone dropdown to ProfilePage using existing Select component patterns
2. **Timezone Search**: Implement searchable timezone list with common timezones prioritized
3. **Save Functionality**: Integrate timezone updates with existing profile save workflow and validation
4. **User Feedback**: Display success/error messages for timezone changes using existing toast patterns
5. **Default Detection**: Suggest user's browser timezone as initial default (optional, not forced)
6. **Visual Integration**: Maintain design consistency with existing currency and locale selectors

#### Integration Verification

- **IV1**: Profile page editing workflow remains functional for all existing fields
- **IV2**: Timezone changes take effect immediately without requiring page refresh
- **IV3**: Profile save operation handles timezone updates alongside other preference changes

### Story 1.4: Apply Timezone to Date Displays

As a user,
I want to see all dates and times displayed in my selected timezone,
so that I can easily understand when my financial activities occurred in my local time.

#### Acceptance Criteria

1. **Transaction Table**: Update `TransactionTable.tsx` to display dates in user timezone while maintaining DD/MM/YYYY format
2. **Transaction Calendar**: Convert `TransactionPage.tsx` calendar and time displays to user timezone
3. **Activity Cards**: Update `RecentActivityCard.tsx` and related components to show timezone-aware timestamps  
4. **Wallet Components**: Convert timestamp displays in wallet-related components (`WalletCard.tsx`, `WalletSummaryCard.tsx`)
5. **Consistent Format**: Maintain existing Vietnamese locale formatting preferences with timezone conversion
6. **Timezone Indicator**: Add optional timezone abbreviation display where space permits (e.g., "GMT+7")

#### Integration Verification

- **IV1**: All existing date filtering and comparison operations continue to function correctly
- **IV2**: Calendar date selection and navigation remains accurate across timezone boundaries
- **IV3**: Performance impact on date-heavy pages like TransactionPage stays within acceptable limits (<100ms)

---

*This PRD ensures comprehensive timezone support while maintaining the integrity and performance of the existing CFPM application. The incremental story approach allows for safe deployment and testing of each component while building toward complete timezone functionality.*
