# Requirements Document

## Introduction

CipherGuard is a comprehensive parental control platform that enables parents to monitor, manage, and protect their children's online activities across devices. The system consists of three main components: a backend REST API (Node.js/Express/MongoDB), a frontend parent dashboard (React/TypeScript), and a browser extension for activity monitoring. This document captures the functional requirements of the existing CipherGuard system.

## Glossary

- **Parent**: An authenticated user who manages and monitors child profiles
- **Child**: A monitored user profile created by a parent, associated with browsing activity
- **Backend_API**: The Node.js/Express REST API that handles authentication, data storage, and business logic
- **Frontend_Dashboard**: The React-based web application used by parents to view and manage child activities
- **Browser_Extension**: The Chrome extension installed on child devices to monitor and control browsing
- **JWT_Token**: JSON Web Token used for authentication between components
- **Extension_Token**: Unique identifier linking a browser extension to a specific child profile
- **Monitored_URL**: A website domain tracked by the system with associated time-spent data
- **Blocked_URL**: A website domain that is restricted and cannot be accessed by the child
- **Incognito_Alert**: A notification generated when incognito/private browsing is detected
- **Activity_Log**: Time-stamped record of browsing behavior including searches and website visits
- **Daily_Time_Spent**: Map of dates to seconds spent on each monitored domain
- **Search_Query**: Text entered into search engines, captured and logged by the extension
- **Heartbeat**: Periodic signal from the extension indicating active monitoring status
- **Session**: A period of active browser usage by a child
- **Category**: Classification of website content (e.g., "general", "social", "education")

## Requirements

### Requirement 1: Parent Authentication

**User Story:** As a parent, I want to create an account and securely log in, so that I can access the monitoring dashboard and manage my children's profiles.

#### Acceptance Criteria

1. WHEN a parent provides name, email, and password, THE Backend_API SHALL create a new parent account with unique email validation
2. WHEN a parent account already exists with the provided email, THE Backend_API SHALL reject the registration and return an error message
3. WHEN a parent provides valid login credentials, THE Backend_API SHALL generate and return a JWT_Token for authenticated access
4. WHEN a parent provides invalid login credentials, THE Backend_API SHALL reject the login attempt and return an error message
5. THE Frontend_Dashboard SHALL store the JWT_Token in browser local storage for subsequent API requests
6. WHEN a parent accesses protected routes, THE Backend_API SHALL verify the JWT_Token before processing requests

### Requirement 2: Child Profile Management

**User Story:** As a parent, I want to create and manage multiple child profiles, so that I can monitor each child's online activity separately.

#### Acceptance Criteria

1. WHEN a parent creates a child profile with name and email, THE Backend_API SHALL generate a unique Extension_Token for that child
2. WHEN a child profile is created, THE Backend_API SHALL link the child to the parent's account
3. THE Backend_API SHALL return the Extension_Token to the Frontend_Dashboard for distribution to the child's device
4. WHEN a parent requests their children list, THE Backend_API SHALL return all child profiles with name, email, and status information
5. THE Frontend_Dashboard SHALL display all monitored child profiles with their current online/offline status
6. WHEN a parent selects a child profile, THE Frontend_Dashboard SHALL display that child's specific activity data

### Requirement 3: Browser Extension Authentication

**User Story:** As a child user, I want to authenticate the browser extension with my token, so that my browsing activity is properly monitored and linked to my profile.

#### Acceptance Criteria

1. WHEN the Browser_Extension is installed, THE Browser_Extension SHALL prompt for an Extension_Token
2. WHEN a valid Extension_Token is provided, THE Browser_Extension SHALL store the JWT_Token in local storage
3. WHEN the Browser_Extension starts, THE Browser_Extension SHALL send an activation notification to the Backend_API
4. WHEN the browser closes or extension is disabled, THE Browser_Extension SHALL send a disconnection notification to the Backend_API
5. THE Browser_Extension SHALL send periodic heartbeat signals to maintain online status
6. WHEN a heartbeat is received, THE Backend_API SHALL update the child's last_heartbeat timestamp and set status to "online"

### Requirement 4: URL Monitoring and Time Tracking

**User Story:** As a parent, I want to track which websites my child visits and how much time they spend on each domain, so that I can understand their online behavior patterns.

#### Acceptance Criteria

1. WHEN a child navigates to a website, THE Browser_Extension SHALL extract the domain name from the URL
2. WHEN the active tab changes or updates, THE Browser_Extension SHALL send the domain to the Backend_API every 60 seconds
3. WHEN a domain is reported, THE Backend_API SHALL add 60 seconds to the Daily_Time_Spent for today's date
4. WHEN a domain is reported for the first time, THE Backend_API SHALL create a new Monitored_URL entry with domain, category, and Daily_Time_Spent map
5. WHEN a domain already exists in monitored URLs, THE Backend_API SHALL increment the time for today's date in the Daily_Time_Spent map
6. THE Backend_API SHALL update the lastUpdated timestamp for each Monitored_URL when time is added
7. WHEN a parent requests web usage statistics, THE Backend_API SHALL calculate total time spent across all domains for today
8. THE Frontend_Dashboard SHALL display total screen time in hours and minutes format

### Requirement 5: Search Query Monitoring

**User Story:** As a parent, I want to see what my child searches for online, so that I can identify potentially concerning interests or behaviors.

#### Acceptance Criteria

1. WHEN a child visits a search engine URL (Google, Bing, Yahoo), THE Browser_Extension SHALL extract the search query from URL parameters
2. WHEN a search query is detected, THE Browser_Extension SHALL send the query text along with the domain to the Backend_API
3. WHEN a search query is received, THE Backend_API SHALL append it to the searchQueries array for that Monitored_URL
4. THE Backend_API SHALL prevent duplicate search queries from being stored in the searchQueries array
5. WHEN a parent requests filtered activity, THE Backend_API SHALL return search queries with timestamps
6. THE Frontend_Dashboard SHALL display search queries separately from regular website visits in the activity monitor

### Requirement 6: Incognito Mode Detection and Alerts

**User Story:** As a parent, I want to be notified when my child attempts to use incognito/private browsing mode, so that I can address attempts to hide online activity.

#### Acceptance Criteria

1. WHEN an incognito window is opened, THE Browser_Extension SHALL detect the incognito window creation event
2. WHEN incognito mode is detected, THE Browser_Extension SHALL immediately close the incognito window
3. WHEN incognito mode is detected, THE Browser_Extension SHALL send an incognito alert with URL and timestamp to the Backend_API
4. WHEN an incognito alert is received, THE Backend_API SHALL append it to the child's incognitoAlerts array
5. WHEN a parent requests alerts, THE Backend_API SHALL return all incognito alerts sorted by timestamp (newest first)
6. THE Frontend_Dashboard SHALL display the count of incognito alerts in the dashboard statistics
7. WHEN a parent views alert details, THE Frontend_Dashboard SHALL show the URL and timestamp for each alert
8. WHEN a parent clears alerts, THE Backend_API SHALL empty the incognitoAlerts array for that child

### Requirement 7: URL Blocking and Unblocking

**User Story:** As a parent, I want to block specific websites and prevent my child from accessing them, so that I can protect them from inappropriate content.

#### Acceptance Criteria

1. WHEN a parent blocks a URL for a child, THE Backend_API SHALL add the URL to the child's blockedUrls array
2. WHEN a URL is already in the blockedUrls array, THE Backend_API SHALL not add duplicate entries
3. WHEN a parent unblocks a URL, THE Backend_API SHALL remove the URL from the child's blockedUrls array
4. WHEN the Browser_Extension navigates to a new URL, THE Browser_Extension SHALL send the URL to the Backend_API for blocking check
5. WHEN a URL check is received, THE Backend_API SHALL compare the URL against all entries in the child's blockedUrls array
6. WHEN a URL matches a blocked entry, THE Backend_API SHALL return blocked status as true
7. WHEN a blocked URL is detected, THE Browser_Extension SHALL display an alert message to the user
8. WHEN a blocked URL is detected, THE Browser_Extension SHALL close the tab immediately
9. THE Frontend_Dashboard SHALL display the count of blocked URLs in the dashboard statistics
10. WHEN a parent views blocked content details, THE Frontend_Dashboard SHALL list all blocked domains

### Requirement 8: Content Filtering (Text and Images)

**User Story:** As a child user, I want offensive content to be automatically filtered from web pages, so that I have a safer browsing experience.

#### Acceptance Criteria

1. THE Browser_Extension SHALL maintain a list of offensive words in multiple languages
2. WHEN a web page loads, THE Browser_Extension SHALL scan all text nodes in the DOM
3. WHEN offensive words are detected in text, THE Browser_Extension SHALL replace them with asterisks
4. WHEN new content is added to the page, THE Browser_Extension SHALL scan and filter the new content
5. WHEN an image has an alt attribute containing offensive words, THE Browser_Extension SHALL apply a blur filter to the image
6. THE Browser_Extension SHALL mark processed images to avoid redundant checks
7. WHEN an image loads, THE Browser_Extension SHALL optionally send it to an NSFW classification API
8. WHEN an image receives an NSFW score above the threshold (0.3), THE Browser_Extension SHALL apply a blur filter to the image

### Requirement 9: Activity Timeline and Filtering

**User Story:** As a parent, I want to view a chronological timeline of my child's online activities with filtering options, so that I can review their behavior over different time periods.

#### Acceptance Criteria

1. WHEN a parent selects a time frame (today, yesterday, week, month), THE Frontend_Dashboard SHALL send the time frame to the Backend_API
2. WHEN a filtered activity request is received, THE Backend_API SHALL filter Monitored_URLs by lastUpdated date within the time frame
3. THE Backend_API SHALL return activities sorted by timestamp in descending order (newest first)
4. THE Backend_API SHALL include both search queries and website visits in the activity list
5. THE Frontend_Dashboard SHALL display activities with appropriate icons (search icon for queries, link icon for websites)
6. THE Frontend_Dashboard SHALL show the timestamp for each activity in human-readable format
7. THE Frontend_Dashboard SHALL refresh activity data every 5 seconds for real-time updates

### Requirement 10: Dashboard Statistics and Analytics

**User Story:** As a parent, I want to see summary statistics about my child's online activity, so that I can quickly assess their screen time and safety metrics.

#### Acceptance Criteria

1. WHEN a parent views the dashboard, THE Frontend_Dashboard SHALL display total screen time for today
2. WHEN a parent views the dashboard, THE Frontend_Dashboard SHALL display the count of incognito alerts
3. WHEN a parent views the dashboard, THE Frontend_Dashboard SHALL display the count of blocked content attempts
4. WHEN a parent views the dashboard, THE Frontend_Dashboard SHALL display the number of active sessions
5. THE Frontend_Dashboard SHALL show percentage changes for each metric compared to previous periods
6. THE Frontend_Dashboard SHALL refresh statistics every 5 seconds
7. WHEN detailed web usage is requested, THE Backend_API SHALL return per-domain time spent with category and last updated information

### Requirement 11: Report Generation and Export

**User Story:** As a parent, I want to generate and download PDF reports of my child's activity, so that I can keep records or share them with other caregivers.

#### Acceptance Criteria

1. WHEN a parent clicks generate report, THE Frontend_Dashboard SHALL fetch web usage, alerts, blocked content, and activity data
2. THE Frontend_Dashboard SHALL compile the data into a formatted PDF document
3. THE PDF report SHALL include child name, email, parent name, and generation date
4. THE PDF report SHALL include total time spent and per-domain usage details
5. THE PDF report SHALL include all incognito alerts with URLs and timestamps
6. THE PDF report SHALL include all blocked URLs
7. THE PDF report SHALL include recent activity timeline
8. WHEN the PDF is generated, THE Frontend_Dashboard SHALL trigger a browser download
9. WHEN report generation fails, THE Frontend_Dashboard SHALL display an error message

### Requirement 12: Real-time Notifications

**User Story:** As a parent, I want to receive real-time notifications about important events, so that I can respond quickly to concerning activities.

#### Acceptance Criteria

1. WHEN a search query is detected, THE Backend_API SHALL send a Telegram notification to the parent
2. WHEN an incognito alert occurs, THE Backend_API SHALL send a Telegram notification to the parent
3. WHEN the Browser_Extension is activated, THE Backend_API SHALL send a Telegram notification to the parent
4. WHEN the Browser_Extension is disconnected, THE Backend_API SHALL send a Telegram notification to the parent
5. WHEN a URL is blocked by the parent, THE Backend_API SHALL send a Telegram notification to the parent
6. THE Backend_API SHALL include child name and relevant details in each notification message
7. WHEN a parent's Telegram chat ID is not configured, THE Backend_API SHALL log the notification without sending

### Requirement 13: User Interface and Navigation

**User Story:** As a parent, I want an intuitive dashboard interface with easy navigation, so that I can efficiently monitor and manage my children's online activities.

#### Acceptance Criteria

1. THE Frontend_Dashboard SHALL display a navigation bar with links to dashboard, login, signup, and contact pages
2. THE Frontend_Dashboard SHALL show the parent's name and email in the profile section
3. THE Frontend_Dashboard SHALL list all monitored child profiles with selection capability
4. WHEN a child profile is selected, THE Frontend_Dashboard SHALL update all statistics and activity views for that child
5. THE Frontend_Dashboard SHALL provide view options for overview, protection settings, and profile management
6. THE Frontend_Dashboard SHALL display a notification bell icon with unread count badge
7. WHEN the notification bell is clicked, THE Frontend_Dashboard SHALL show a dropdown with recent notifications
8. THE Frontend_Dashboard SHALL provide a settings menu with account settings, security, and logout options
9. WHEN logout is clicked, THE Frontend_Dashboard SHALL clear the JWT_Token and redirect to the login page
10. THE Frontend_Dashboard SHALL provide an "Add Profile" button to navigate to the child creation page

### Requirement 14: Protection Settings Management

**User Story:** As a parent, I want to configure protection settings for content filtering and time restrictions, so that I can customize the level of monitoring for my child.

#### Acceptance Criteria

1. THE Frontend_Dashboard SHALL display toggles for content filtering options (adult content, violence, profanity)
2. THE Frontend_Dashboard SHALL display toggles for time restriction options (school hours, bedtime, weekend limits)
3. THE Frontend_Dashboard SHALL display AI protection level sliders for image analysis, text analysis, and behavioral analysis
4. WHEN protection settings are modified, THE Frontend_Dashboard SHALL provide a save button to persist changes
5. THE Frontend_Dashboard SHALL provide a reset button to restore default protection settings
6. WHEN settings are saved, THE Frontend_Dashboard SHALL display a confirmation toast message

### Requirement 15: Device Status and Heartbeat Monitoring

**User Story:** As a parent, I want to see which of my children's devices are currently online and protected, so that I know when monitoring is active.

#### Acceptance Criteria

1. WHEN the Browser_Extension sends a heartbeat, THE Backend_API SHALL update the child's lastHeartbeat timestamp
2. WHEN a heartbeat is received, THE Backend_API SHALL set the child's status to "online"
3. WHEN the Browser_Extension disconnects, THE Backend_API SHALL set the child's status to "offline"
4. THE Frontend_Dashboard SHALL display each child's status as "online" or "offline"
5. WHEN a child is online, THE Frontend_Dashboard SHALL show a green pulsing indicator
6. THE Frontend_Dashboard SHALL display "Active & Protected" status for online children
7. THE Frontend_Dashboard SHALL display the Extension_Token for each child profile
