# UX Guidelines

This document outlines the user experience principles and patterns we follow across the application.

## Core Principles

### 1. Consistency
- Use the same interaction patterns throughout the application
- Maintain consistent visual hierarchy and spacing
- Follow established component patterns and props interfaces

### 2. Simplicity
- Minimize cognitive load for users
- Present information in logical, progressive disclosure
- Avoid unnecessary complexity in interactions

### 3. Feedback
- Provide immediate visual feedback for user actions
- Show loading states during async operations  
- Display clear success and error messages
- Indicate system status to users

## Form Design Patterns

### Action Detection
- **Pattern**: Detect create vs edit mode by entity presence, not explicit mode props
- **Implementation**: `const action = entity ? updateEntity : createEntity`
- **Benefits**: Cleaner component interfaces, less prop drilling

### Search Components
- **Pattern**: Use searchable dropdowns instead of static Select components for user selection
- **Implementation**: UserSearch component with real database queries
- **Benefits**: Scales with large datasets, better user experience

### Error Handling
- **Visual Indicators**: Use `hasError` props to highlight invalid fields
- **Error Messages**: Display field-specific error messages below inputs
- **Form-level Errors**: Show general errors in Callout components at top of form

### Date Selection
- **Pattern**: Use Tremor DatePicker components instead of native date inputs
- **Features**: Visual calendar interface, date validation, consistent styling
- **Implementation**: Controlled components with hidden form fields for submission

## Component Architecture

### Generic Components
- **Principle**: Build reusable components that work with any data type
- **Example**: GenericSearch component that can be specialized for users, projects, etc.
- **Benefits**: Consistency, maintainability, reduced code duplication

### Hidden Form Fields
- **Pattern**: Use hidden inputs for IDs instead of complex action wrappers
- **Implementation**: `{entity && <input type="hidden" name="entityId" value={entity.id} />}`
- **Benefits**: Cleaner action logic, simpler form submission

### Focus Management
- **Pattern**: Keep input focus during search interactions
- **Implementation**: Work with Radix UI's focus management, not against it
- **Avoid**: setTimeout focus hacks, complex focus state tracking

## State Management

### Form State
- **Pattern**: Use union types for create/update form states
- **Implementation**: `type FormState = CreateState | UpdateState`
- **Benefits**: Type safety, consistent error handling

### Component State
- **Principle**: Minimize state complexity, prefer derived state
- **Example**: `const showResults = searchResults.length > 0 && !isSelected`
- **Avoid**: Redundant state that can be computed from existing state

## Loading & Async States

### Search Debouncing
- **Default**: 300ms debounce for search inputs
- **Minimum Length**: 2 characters before triggering search
- **Visual Feedback**: Show spinner during search operations

### Form Submissions
- **Loading States**: Disable submit buttons during submission
- **Success Feedback**: Show success message before redirecting
- **Error Recovery**: Keep form state on error, allow retry

## Accessibility

### Keyboard Navigation
- **Search Components**: Support arrow keys, enter, and escape
- **Focus Management**: Maintain logical tab order
- **Screen Readers**: Use proper ARIA labels and descriptions

### Error States
- **Visual**: Use color and icons together (not just color)
- **Semantic**: Use proper ARIA attributes for invalid states
- **Clear Messages**: Provide actionable error descriptions

## Performance

### Component Optimization
- **Lazy Loading**: Load heavy components only when needed
- **Debouncing**: Prevent excessive API calls in search
- **Memoization**: Use React.memo for expensive computations when appropriate

### Database Queries
- **Pagination**: Limit search results (default: 10 items)
- **Field Selection**: Only select needed fields in queries
- **Indexing**: Ensure search fields are properly indexed

---

## Contributing to UX Guidelines

When adding new patterns or updating existing ones:
1. Document the problem the pattern solves
2. Provide concrete implementation examples
3. Explain the benefits and trade-offs
4. Update related components to follow new patterns
5. Add accessibility considerations where relevant

This document evolves with our application. Update it as we discover new patterns or improve existing ones.

---

# Post-Registration User Experience Flow

## Current State Analysis
The system currently has:
- **Admin Section**: Full CRUD for users, organisations, teams, projects (admin-only access)
- **Basic User Interface**: Landing page, file upload functionality, basic account settings
- **User Roles**: GUEST, CUSTOMER, ADMIN (new users default to CUSTOMER role)
- **Complete Database Schema**: Organizations, teams, team members, projects with proper relationships

## Proposed User Experience Flow

### 1. Welcome/Onboarding Flow (`/dashboard/welcome`)
**First-time login redirect for CUSTOMER users**
- Welcome screen explaining the IPFS storage platform
- Step-by-step setup wizard:
  1. **Create Organization** (user becomes owner)
  2. **Create Team** (user becomes team lead) 
  3. **Add Team Members** (optional - invite by email)
  4. **Create First Project**
  5. **Set Storage Allocation** (mocked values)

### 2. Main Dashboard (`/dashboard`)
**Central hub for all user activities**
- **Overview Cards**: Active projects, storage usage, team members
- **Quick Actions**: Create project, upload files, invite members
- **Recent Activity**: Latest uploads, project updates
- **Storage Metrics**: Used/allocated space, file counts (mocked initially)

### 3. Organization Management (`/dashboard/organization`)
**Organization-level management for organization owners**
- View/edit organization details
- Member management (view all members across teams)
- Organization-wide settings and storage limits
- Billing/subscription info (mocked)

### 4. Team Management (`/dashboard/teams`)
**Team collaboration features**
- List user's teams (owned + member of)
- Create new teams
- Manage team members and permissions
- Team-specific storage quotas
- Team activity and project overview

### 5. Project Management (`/dashboard/projects`)
**Enhanced project interface beyond admin CRUD**
- **Project Dashboard**: Storage usage, file counts, activity
- **File Management**: Upload, organize, and manage IPFS files
- **Storage Allocation**: Set and monitor storage limits per project
- **Team Collaboration**: Share access, assign tasks
- **Project Settings**: Repository links, documentation, team assignment

### 6. File & Storage Management (`/dashboard/files`)
**IPFS-focused file management**
- **File Browser**: Navigate project files with IPFS hashes
- **Upload Interface**: Improved from current `/account/upload`
- **Storage Analytics**: Usage charts, file type breakdown
- **Pinning Management**: Pin/unpin files, replication status
- **Share Files**: Generate shareable IPFS gateway links

### 7. Navigation & Layout
**Consistent dashboard layout**
- **Sidebar Navigation**: Dashboard, Projects, Teams, Organization, Files, Settings
- **Breadcrumb Navigation**: Clear hierarchy
- **User Context Switching**: Switch between organizations if user belongs to multiple
- **Notifications**: System alerts, invitations, storage warnings

## Implementation Strategy

### Phase 1: Core Dashboard Structure
1. Create dashboard layout with sidebar navigation
2. Implement welcome/onboarding flow with organization setup
3. Build main dashboard with overview cards
4. Add routing and middleware for CUSTOMER role access

### Phase 2: Organization & Team Management
1. Create organization management pages (reusing existing forms)
2. Build team management interface
3. Implement team member invitation system
4. Add user context switching for multi-organization users

### Phase 3: Enhanced Project Management
1. Create project dashboard with storage metrics (mocked)
2. Build project-specific file management
3. Add storage allocation controls
4. Integrate with existing IPFS upload functionality

### Phase 4: Advanced Features
1. Implement file browser with IPFS integration
2. Add storage analytics and reporting
3. Create notification system
4. Build activity feeds and collaboration features

## Technical Approach

### New Pages Structure
```
/dashboard/
├── welcome/                 # Onboarding wizard
├── page.tsx                # Main dashboard
├── organization/           # Organization management
├── teams/                  # Team management
├── projects/               # Enhanced project views
├── files/                  # File management
└── layout.tsx              # Dashboard shell
```

### Components Strategy
- **Reuse Existing**: Forms, tables, and components from admin section
- **Enhance UX**: Add wizards, progress indicators, better visualization
- **Add Analytics**: Charts for storage usage (using existing Recharts)
- **Mock Storage Data**: Initially simulate storage metrics, prepare for real IPFS integration

### Data Flow
1. **User Registration** → Redirects to `/dashboard/welcome`
2. **Welcome Wizard** → Creates organization → Creates team → Creates project
3. **Dashboard** → Shows user's context (organizations, teams, projects)
4. **File Operations** → Integrates with existing IPFS cluster API

## User Experience Principles for Dashboard
- **Progressive Disclosure**: Start simple, reveal complexity as needed
- **Context Awareness**: Show relevant information based on user's role and memberships
- **Guided Experience**: Clear onboarding and help throughout
- **Consistent Patterns**: Follow established UX guidelines and component patterns
- **Performance**: Efficient queries and caching for dashboard data

## Dashboard-Specific UX Patterns

### Onboarding Wizard
- **Pattern**: Multi-step wizard with progress indicator
- **Implementation**: Step-by-step forms with validation before proceeding
- **Benefits**: Guides users through complex setup, reduces abandonment

### Context Switching
- **Pattern**: Organization/team selector in header
- **Implementation**: Dropdown with user's organizations and current context
- **Benefits**: Clear indication of current scope, easy switching

### Storage Visualization
- **Pattern**: Progress bars and charts for storage usage
- **Implementation**: Recharts components with percentage calculations
- **Benefits**: Clear understanding of usage limits and availability

### Quick Actions
- **Pattern**: Prominent action buttons on dashboard cards
- **Implementation**: Card-based layout with primary actions
- **Benefits**: Fast access to common tasks, discoverability