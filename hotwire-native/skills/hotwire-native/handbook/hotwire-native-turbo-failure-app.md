# Hotwire Native Authentication with TurboFailureApp
## Implementing Proper 4xx Response Codes for Mobile Apps

### Table of Contents
1. [The Problem with Standard Devise Authentication](#the-problem)
2. [TurboFailureApp Implementation](#turbofailureapp-implementation)
3. [Authentication Flow Architecture](#authentication-flow-architecture)
4. [Mobile App Integration](#mobile-app-integration)
5. [Configuration Setup](#configuration-setup)
6. [Testing Strategies](#testing-strategies)
7. [Troubleshooting Guide](#troubleshooting-guide)

---

## The Problem

### Standard Web Authentication vs Mobile App Requirements

**Web Browser Behavior:**
- Unauthenticated requests receive `302 Found` redirects to login page
- Browsers automatically follow redirects and display the login form
- CSRF tokens and session cookies work seamlessly

**Mobile App (Hotwire Native) Requirements:**
- Apps need explicit HTTP status codes to determine authentication state
- `401 Unauthorized` signals "authentication required"
- `422 Unprocessable Entity` indicates "authentication failed"
- Redirects can cause navigation confusion in native app contexts

### Default Devise Behavior
```ruby
# Standard Devise FailureApp behavior
class Devise::FailureApp
  def respond
    if http_auth?
      http_auth
    else
      redirect  # Returns 302 redirect - problematic for mobile apps
    end
  end
end
```

**Problems with Standard Approach:**
1. Mobile apps receive unexpected redirects instead of clear error codes
2. Authentication state becomes ambiguous
3. Native navigation can break when following web-based redirects
4. API endpoints return HTML instead of JSON error responses

---

## TurboFailureApp Implementation

### Core Implementation
```ruby
# config/initializers/devise.rb
class TurboFailureApp < Devise::FailureApp
  # Compatibility for Turbo::Native::Navigation
  class << self
    def helper_method(*methods)
    end
  end

  include Turbo::Native::Navigation

  # Intercept for Hotwire Native:
  # Return a 401 for any :authenticate_user before actions
  # Return a 422 for any login failures
  #
  # This param is set in a before_action on Devise controllers to ensure they don't return 401s
  def http_auth?
    (hotwire_native_app? && !params["hotwire_native_form"]) || super
  end
end
```

### Key Method Breakdown

#### `http_auth?` Method
**Purpose**: Determines whether to return HTTP status codes vs redirects

**Logic Flow**:
1. **Hotwire Native Detection**: Check if request comes from mobile app
2. **Form Parameter Check**: Look for `hotwire_native_form` parameter
3. **Fallback**: Use standard Devise behavior for web requests

```ruby
def http_auth?
  # Simplified logic: Return HTTP auth for native apps unless it's a form submission
  (hotwire_native_app? && !params["hotwire_native_form"]) || super
end
```

**Key Changes from Standard Implementation**:
- Uses concise boolean logic instead of conditional blocks
- Relies on `Turbo::Native::Navigation`'s built-in `hotwire_native_app?` detection
- No custom Android handling (uses Turbo Native's default behavior)

---

## Authentication Flow Architecture

### Request Flow Diagram
```
Mobile App Request
        ↓
   Authentication Required?
        ↓              ↓
      YES             NO
        ↓              ↓
  TurboFailureApp   Continue
        ↓              ↓
  hotwire_native_app?  Response
        ↓       ↓
       YES     NO
        ↓       ↓
   Return 401  Return 302
   (HTTP Auth) (Redirect)
```

### Response Code Mapping
| Scenario | Web Response | Mobile Response | Purpose |
|----------|-------------|-----------------|---------|
| Not authenticated | `302 → /login` | `401 Unauthorized` | Prompt login |
| Login failed | `302 → /login` | `422 Unprocessable Entity` | Show error |
| Invalid credentials | `302 → /login` | `422 Unprocessable Entity` | Validation failed |
| Session expired | `302 → /login` | `401 Unauthorized` | Re-authenticate |
| Form submission | `302 → /login` | `302 → /login` | Allow normal flow |

### Controller Integration

#### App Namespace Protection
```ruby
# app/controllers/app_controller.rb
class AppController < ApplicationController
  before_action :authenticate_user!  # Triggers TurboFailureApp on failure
  layout "app"
  respond_to :html, :json
end
```

#### Custom Sessions Controller
```ruby
# app/controllers/users/sessions_controller.rb
class Users::SessionsController < Devise::SessionsController
  skip_before_action :verify_authenticity_token, only: [:new, :create]
  respond_to :html, :json

  def create
    respond_to do |format|
      format.html { super }
      format.json do
        self.resource = warden.authenticate(auth_options)
        if resource
          resource.remember_me = true
          sign_in(resource_name, resource)
          render json: {
            message: "User signed in successfully.",
            user: { id: resource.id, email: resource.email },
            redirect_url: after_sign_in_path_for(resource)
          }, status: :ok
        else
          render json: { error: "Invalid email or password" }, status: :unauthorized
        end
      end
    end
  end

  def failure
    respond_to do |format|
      format.html { super }
      format.json do
        render json: { error: "Invalid email or password" }, status: :unauthorized
      end
    end
  end
end
```

---

## Mobile App Integration

### iOS Hotwire Native Implementation

#### Authentication State Detection
```swift
// iOS Native Code
func handleResponse(_ response: HTTPURLResponse) {
    switch response.statusCode {
    case 401:
        // User needs to authenticate
        presentLoginScreen()
    case 422:
        // Authentication failed, show error
        showAuthenticationError()
    case 200...299:
        // Success, continue normal flow
        handleSuccessResponse()
    default:
        // Handle other status codes
        handleError(response.statusCode)
    }
}
```

#### Turbo Session Configuration
```swift
// Configure Turbo session to handle authentication
session.delegate = self

extension ViewController: SessionDelegate {
    func session(_ session: Session, didFailRequestForVisitable visitable: Visitable, error: Error) {
        if let turboError = error as? TurboError,
           case let .http(statusCode) = turboError {
            switch statusCode {
            case 401:
                presentAuthenticationController()
            case 422:
                showValidationErrors()
            default:
                showErrorMessage(for: statusCode)
            }
        }
    }
}
```

### Form Submission Handling

#### Hotwire Native Form Parameter
```erb
<!-- app/views/devise/sessions/new.html.erb -->
<%= form_with(model: resource, 
              as: resource_name, 
              url: session_path(resource_name), 
              local: false,
              data: { turbo: true }) do |f| %>
  
  <!-- Add hidden field for native apps -->
  <%= hidden_field_tag :hotwire_native_form, "true" if hotwire_native_app? %>
  
  <div class="field">
    <%= f.label :email %>
    <%= f.email_field :email, autofocus: true, autocomplete: "email" %>
  </div>

  <div class="field">
    <%= f.label :password %>
    <%= f.password_field :password, autocomplete: "current-password" %>
  </div>

  <div class="actions">
    <%= f.submit "Log in" %>
  </div>
<% end %>
```

#### Helper Method for Detection
```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def hotwire_native_app?
    request.user_agent&.include?("Turbo Native")
  end
end
```

---

## Configuration Setup

### Complete Devise Configuration
```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  # Configure mailer
  config.mailer_sender = "Your App <hello@yourapp.com>"
  
  # Configure navigational formats to include JSON
  config.navigational_formats = ["*/*", :html, :turbo_stream, :json]
  
  # Hotwire/Turbo configuration
  config.responder.error_status = :unprocessable_entity
  config.responder.redirect_status = :see_other
  
  # Configure Warden to use TurboFailureApp
  config.warden do |manager|
    manager.failure_app = TurboFailureApp
  end
  
  # OAuth configuration
  config.omniauth :google_oauth2, ENV['GOOGLE_CLIENT_ID'], ENV['GOOGLE_CLIENT_SECRET']
  config.omniauth :apple, ENV['APPLE_CLIENT_ID'], "", {
    scope: "email name",
    team_id: ENV['APPLE_TEAM_ID'],
    key_id: ENV['APPLE_KEY_ID'],
    pem: ENV['APPLE_PEM']
  }
end
```

### Route Configuration
```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: {
    registrations: "users/registrations",
    omniauth_callbacks: "users/omniauth_callbacks",
    sessions: "users/sessions"
  }

  # Custom auth endpoints for mobile apps
  devise_scope :user do
    post "users/apple_auth", to: "users/apple_auth#authenticate_apple_user"
    post "users/google_auth", to: "users/google_auth#authenticate_google_user"
  end

  # Protected app routes
  namespace :app do
    resources :dashboard, only: [:index]
    # ... other protected routes
  end
end
```

### Environment-Specific Settings
```ruby
# config/environments/development.rb
Rails.application.configure do
  # Allow insecure HTTP for development mobile testing
  config.force_ssl = false
  
  # Enable detailed logging for authentication debugging
  config.log_level = :debug
end

# config/environments/production.rb
Rails.application.configure do
  # Ensure secure connections for production
  config.force_ssl = true
  
  # Configure proper CORS for mobile apps
  config.middleware.insert_before 0, Rack::Cors do
    allow do
      origins '*'
      resource '*', 
        headers: :any, 
        methods: [:get, :post, :put, :patch, :delete, :options, :head],
        expose: ['X-CSRF-Token']
    end
  end
end
```

---

## Testing Strategies

### RSpec Integration Tests
```ruby
# spec/requests/authentication_spec.rb
RSpec.describe "Authentication", type: :request do
  describe "TurboFailureApp integration" do
    context "when request comes from Hotwire Native iOS app" do
      let(:headers) { { "User-Agent" => "Turbo Native iOS" } }
      
      it "returns 401 for unauthenticated requests" do
        get "/app/dashboard", headers: headers
        expect(response).to have_http_status(:unauthorized)
        expect(response.content_type).to include("application/json")
      end
    end

    context "when request comes from web browser" do
      let(:headers) { { "User-Agent" => "Mozilla/5.0" } }
      
      it "redirects to login page" do
        get "/app/dashboard", headers: headers
        expect(response).to have_http_status(:found)
        expect(response).to redirect_to(new_user_session_path)
      end
    end

    context "when request comes from Android" do
      let(:headers) { { "User-Agent" => "Turbo Native Android" } }
      
      it "uses Turbo Native default behavior" do
        get "/app/dashboard", headers: headers
        # Behavior depends on Turbo::Native::Navigation implementation
        expect(response).to have_http_status(:unauthorized).or have_http_status(:found)
      end
    end
  end

  describe "Login failures" do
    context "with invalid credentials from mobile app" do
      let(:headers) { { "User-Agent" => "Turbo Native iOS" } }
      
      it "returns 422 with JSON error" do
        post "/users/sign_in", 
             params: { user: { email: "test@example.com", password: "wrong" } },
             headers: headers.merge({ "Accept" => "application/json" })
             
        expect(response).to have_http_status(:unprocessable_entity)
        expect(JSON.parse(response.body)).to include("error")
      end
    end
  end
end
```

### Feature Testing with User Agents
```ruby
# spec/features/mobile_authentication_spec.rb
RSpec.describe "Mobile Authentication", type: :feature, js: true do
  before do
    page.driver.header("User-Agent", "Turbo Native iOS")
  end

  scenario "unauthenticated mobile user receives proper status codes" do
    visit "/app/dashboard"
    
    # Expect proper status code handling in mobile context
    expect(page.status_code).to eq(401)
  end
end
```

### Manual Testing Checklist
```bash
# Test with curl commands

# 1. Test unauthenticated request from iOS
curl -H "User-Agent: Turbo Native iOS" \
     -v http://localhost:3000/app/dashboard
# Expected: HTTP/1.1 401 Unauthorized

# 2. Test unauthenticated request from web
curl -H "User-Agent: Mozilla/5.0" \
     -v http://localhost:3000/app/dashboard
# Expected: HTTP/1.1 302 Found (redirect)

# 3. Test failed login from mobile
curl -X POST \
     -H "User-Agent: Turbo Native iOS" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d '{"user":{"email":"test@example.com","password":"wrong"}}' \
     -v http://localhost:3000/users/sign_in
# Expected: HTTP/1.1 422 Unprocessable Entity
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### Issue: Mobile App Still Receives Redirects
**Symptoms**: 302 responses instead of 401/422
**Diagnosis**:
```ruby
# Add debugging to TurboFailureApp
def http_auth?
  Rails.logger.info "User Agent: #{request.user_agent}"
  Rails.logger.info "Hotwire Native?: #{hotwire_native_app?}"
  Rails.logger.info "Params: #{params.inspect}"
  
  result = if hotwire_native_app?
    return true unless params["hotwire_native_form"]
  else
    super
  end
  
  Rails.logger.info "HTTP Auth Result: #{result}"
  result
end
```

**Solutions**:
1. Verify User-Agent string detection matches Turbo::Native::Navigation expectations
2. Check if `Turbo::Native::Navigation` is properly included
3. Ensure mobile app sends correct User-Agent header

#### Issue: Form Submissions Return Wrong Status Codes
**Symptoms**: Forms in mobile app return 401 instead of processing
**Diagnosis**: `hotwire_native_form` parameter missing or incorrect

**Solution**:
```erb
<!-- Ensure parameter is included in forms -->
<%= form_with(model: resource, local: false) do |f| %>
  <%= hidden_field_tag :hotwire_native_form, "true" if hotwire_native_app? %>
  <!-- rest of form -->
<% end %>
```

#### Issue: JSON Responses Not Returned
**Symptoms**: HTML returned instead of JSON for mobile requests
**Solutions**:
1. Add `Accept: application/json` header in mobile requests
2. Configure controllers to respond to JSON:
```ruby
class ApplicationController < ActionController::Base
  respond_to :html, :json
end
```

#### Issue: CSRF Token Errors
**Symptoms**: `ActionController::InvalidAuthenticityToken` errors
**Solutions**:
```ruby
# Skip CSRF for API endpoints
class Users::SessionsController < Devise::SessionsController
  skip_before_action :verify_authenticity_token, only: [:create]
end
```

### Debugging Tools

#### Request Logger
```ruby
# config/application.rb
class Application < Rails::Application
  if Rails.env.development?
    config.middleware.use Class.new do
      def initialize(app)
        @app = app
      end

      def call(env)
        request = ActionDispatch::Request.new(env)
        Rails.logger.info "=== REQUEST DEBUG ==="
        Rails.logger.info "User-Agent: #{request.user_agent}"
        Rails.logger.info "Accept: #{request.headers['Accept']}"
        Rails.logger.info "Path: #{request.path}"
        Rails.logger.info "Method: #{request.method}"
        
        status, headers, response = @app.call(env)
        
        Rails.logger.info "Response Status: #{status}"
        Rails.logger.info "===================="
        
        [status, headers, response]
      end
    end
  end
end
```

### Performance Considerations

#### Caching User Agent Detection
```ruby
class TurboFailureApp < Devise::FailureApp
  # Note: hotwire_native_app? comes from Turbo::Native::Navigation
  # No need to override unless you have specific platform requirements
end
```

#### Minimal Overhead
- User-Agent string parsing is fast
- No database queries required
- Caching prevents repeated string operations

This implementation provides a robust foundation for handling authentication in Hotwire Native applications while maintaining backward compatibility with web browsers. The key insight is distinguishing between authentication challenges (401) and authentication failures (422), allowing mobile apps to respond appropriately to each scenario.