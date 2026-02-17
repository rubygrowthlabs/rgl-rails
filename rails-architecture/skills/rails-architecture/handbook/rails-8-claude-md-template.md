# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is a Rails 8 web application. [Customize this description for your specific application.]

**Core Technologies**: Rails 8.0.2, Ruby 3.4.1, SQLite3, Tailwind CSS, Stimulus, Turbo (Hotwire), Solid Queue, Devise, Kamal deployment.

## Development Commands

### Essential Commands
- `bin/dev` - Start development server (includes Rails server and Tailwind CSS watching)
- `bin/rails server` - Start Rails server only
- `bin/rails tailwindcss:watch` - Watch and compile Tailwind CSS
- `bundle install` - Install Ruby dependencies
- `bundle exec rails db:migrate` - Run database migrations
- `bundle exec rails db:seed` - Seed the database with initial data

### Testing
- `bundle exec rspec` - Run all RSpec tests (expect 80% test coverage)
- `bundle exec rspec spec/models/` - Run model tests only
- `bundle exec rspec spec/controllers/` - Run controller tests only
- `bundle exec rspec spec/system/` - Run system/integration tests
- `COVERAGE=true bundle exec rspec` - Run tests with coverage report

### Code Quality
- `bundle exec rubocop` - Run RuboCop linter (Rails Omakase configuration)
- `bundle exec brakeman` - Run security analysis

### Database Management
- `bin/rails db:create` - Create databases
- `bin/rails db:drop` - Drop databases
- `bin/rails db:reset` - Drop, create, migrate, and seed database
- `bin/rails console` - Open Rails console for debugging

## Architecture Overview

### Core Application Structure
This is a Rails 8 application following standard Rails conventions with modern enhancements.

**Key Models:**
- `User` - Authentication and user management (Devise)
- [Add your application-specific models here]

**Background Job Architecture:**
- Uses Solid Queue for background job processing
- [Customize job descriptions for your application]

### Frontend Architecture
- **CSS Framework**: Tailwind CSS with live compilation
- **JavaScript**: Stimulus controllers with Turbo (Hotwire) for SPA-like experience
- **Asset Pipeline**: Propshaft for modern asset management
- **Real-time Updates**: Turbo Streams for live updates
- **No Node.js/NPM/Yarn**: Pure Rails asset pipeline approach

### Authentication & Authorization
- Uses Devise for authentication with custom controllers in `app/controllers/users/`
- Custom sign-in, registration, and password reset pages
- Admin functionality controlled via `admin` boolean on User model

### Key Controllers
- [Add your application-specific controllers here]

### Database Configuration
- Development: SQLite3 with separate databases for cache, queue, and cable
- Uses Solid Cache, Solid Queue, and Solid Cable for Rails 8 features
- Active Storage for file attachments

### Deployment Architecture
- Containerized with Docker
- Deployed using Kamal to Ubuntu servers
- SSL via Let's Encrypt
- Production domain: [your-domain.com]
- Docker registry: [your-registry/your-app]

## Coding Standards

### Rails Conventions
- Follow Rails 8 conventions with modern enhancements
- Use `ApplicationController` as base with `allow_browser versions: :modern`
- Implement custom Devise controllers in `app/controllers/users/` for mobile app support
- Protect routes with `authenticate_user!` before_action
- Use strong parameters with `params.expect(model: [...])` pattern
- Keep methods short and focused - avoid long methods
- Practice Test-Driven Development (TDD) - write tests first, then implement

### API Integration Patterns
- Store API keys in Rails credentials: `Rails.application.credentials.dig(:service_name, :api_key)`
- Always use background jobs for external API operations to avoid blocking web requests
- Implement proper error handling for external service failures

### Frontend Standards
- Use Tailwind CSS utility-first approach
- Implement Stimulus controllers for JavaScript interactions
- Use Turbo Streams and Hotwire for real-time updates (prefer over Ajax)
- Follow mobile-first responsive design principles
- Support Hotwire Native for mobile apps with proper authentication patterns

### Database Patterns
- Use SQLite3 with separate databases for cache, queue, and cable
- Implement Active Storage for file attachments
- Use proper associations between models
- Add database indexes for frequently queried columns

### Security Requirements
- **NEVER** commit API keys, tokens, or secrets to repository
- Store sensitive data in Rails credentials: `rails credentials:edit`
- Use environment variables for deployment secrets
- Skip CSRF protection carefully and only when necessary for API endpoints
- Implement proper authentication for both web and mobile clients

## Required before each commit

### Code Quality Checks
```bash
# Run Rails Omakase linting
bundle exec rubocop

# Run security analysis
bundle exec brakeman

# Run full test suite with 80% coverage requirement
bundle exec rspec

# Check for secrets/API keys
git diff --cached | grep -i "api_key\|secret\|token\|password" || echo "No secrets found"
```

### Database Integrity
- Run `bin/rails db:migrate` if schema changes exist
- Verify migrations are reversible with proper `down` methods
- Test migration rollback in development: `bin/rails db:rollback`

### Test Coverage
- Ensure new features have corresponding RSpec tests (80% coverage expected)
- Use FactoryBot for test data generation
- Test both web and mobile app scenarios where applicable
- Verify background jobs work correctly in test environment
- Follow TDD approach - write failing tests first, then implement

## Important File Locations

### Configuration
- `config/credentials.yml.enc` - Encrypted credentials (API keys)
- `config/deploy.yml` - Kamal deployment configuration
- `config/routes.rb` - Application routes
- `Procfile.dev` - Development process configuration

### Key Directories
- `app/jobs/` - Background job classes
- `app/javascript/controllers/` - Stimulus controllers including bridge controllers for mobile app
- `spec/` - RSpec test files
- `app/views/devise/` - Custom authentication views
- `config/initializers/` - App initialization files

### Mobile App Integration
- Bridge controllers in `app/javascript/controllers/bridge/` for mobile app communication
- Configuration endpoints for mobile apps

## Development Workflow

### Real-time Updates
- Uses Turbo Streams for live updates
- Updates both detail and index pages simultaneously
- No polling required - updates pushed via WebSocket connections

### Testing Strategy (TDD Approach)
- Write tests first, then implement features
- RSpec for unit and integration testing
- FactoryBot for test data generation
- Capybara with Selenium for system tests
- Database cleaner for test isolation
- Maintain 80% test coverage

### Daily Development
```bash
# Start development environment
bin/dev  # Runs Rails server + Tailwind CSS watcher

# Alternative: separate processes
bin/rails server
bin/rails tailwindcss:watch
```

### Background Job Processing
- Jobs run in Puma process via `SOLID_QUEUE_IN_PUMA=true` in development
- Monitor job execution: `bin/rails console` → `SolidQueue::Job.all`
- Failed jobs are retained for debugging
- Use `YourJob.perform_later(params)` for async processing

### Mobile App Integration
- Support Hotwire Native apps with proper authentication flow
- Use `hotwire_native_app?` helper for mobile-specific behavior
- Implement `TurboFailureApp` for proper 401/422 status codes
- Add `hotwire_native_form` parameter for form submissions

### Deployment Process
```bash
# Deploy to production using Kamal
bin/kamal setup    # First-time setup
bin/kamal deploy   # Deploy updates

# Check deployment status
bin/kamal app logs
bin/kamal app exec --interactive bash
```

## Environment Setup Requirements

### Required API Keys
- [List your application-specific API keys here]
- Configure in Rails credentials: `rails credentials:edit`

### Development Dependencies
- Ruby 3.4.1
- SQLite3
- ImageMagick (for image processing)

### Production Requirements
- Docker for containerization
- Kamal for deployment
- Ubuntu server with SSL support

## Key Guidelines

### Never Commit Secrets
- API keys belong in `rails credentials:edit`
- Use environment variables for deployment: `ENV['RAILS_MASTER_KEY']`
- Store deployment secrets in `.kamal/secrets/`
- Double-check diffs before committing: `git diff --cached`

### Background Job Best Practices
- Use background jobs for all external API operations
- Implement proper error handling and retry logic
- Keep job payloads simple (pass IDs, not full objects)
- Use `perform_later` for async, `perform_now` only in tests

### Real-time Features
- Always broadcast updates for long-running operations
- Update both detail views and index pages simultaneously
- Use Turbo Streams and Hotwire for seamless user experience
- Handle connection failures gracefully

### Mobile-first Design
- Design for mobile first, then enhance for desktop
- Use Tailwind's responsive prefixes: `sm:`, `md:`, `lg:`
- Test on actual mobile devices when possible
- Support both touch and mouse interactions

### Authentication Patterns
- Web users: standard Devise redirects and session management
- Mobile users: proper HTTP status codes (401, 422) via `TurboFailureApp`
- API endpoints: JSON responses with appropriate status codes
- Always verify user authorization for protected operations

### Performance Considerations
- Use database indexes for frequently queried columns
- Implement proper eager loading: `.includes(:associated_model)`
- Cache expensive operations when appropriate
- Monitor background job queue length in production

### Rails 8 Modern Features
- Use Solid Queue instead of Sidekiq/Resque
- Use Solid Cache for application caching
- Use Solid Cable for ActionCable WebSocket connections
- Leverage modern browser features with `allow_browser versions: :modern`

### Error Handling
- Gracefully handle external service failures
- Provide meaningful error messages to users
- Log errors appropriately (avoid logging API keys)
- Use Rails error reporting for production issues

### Code Organization
- Keep controllers thin, models focused
- Extract complex logic into service objects when needed
- Use concerns for shared behavior
- Organize JavaScript controllers by feature in `app/javascript/controllers/`
- Avoid long methods - break them into smaller, focused methods
