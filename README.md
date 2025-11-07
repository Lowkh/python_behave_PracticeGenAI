# Expert Python Behave Practice Guide for Full Stack Developers

## Overview

This comprehensive guide demonstrates how to build a **production-ready Behavior-Driven Development (BDD) test automation framework** using Python Behave for full stack web applications. You'll learn to test both **web UI (frontend)** and **REST APIs (backend)** using best practices and simplified patterns.

## What is Behave?

**Behave** is Python's BDD framework that lets you write tests in plain English using Gherkin syntax. It bridges the gap between business requirements and technical implementation, making tests readable for both developers and non-technical stakeholders.

**Key Benefits:**
- Tests written in natural language (Given-When-Then)
- Excellent for full stack testing (UI + API)
- Promotes collaboration between teams
- Easy to maintain and scale

---

## Project Structure

Here's the recommended structure for a full stack Behave project:

```
behave-fullstack-project/
│
├── features/                      # Feature files (Gherkin scenarios)
│   ├── web/                       # Web UI feature files
│   │   ├── login.feature
│   │   └── user_profile.feature
│   │
│   ├── api/                       # API feature files
│   │   ├── user_api.feature
│   │   └── product_api.feature
│   │
│   ├── steps/                     # Step definitions (Python code)
│   │   ├── web_steps.py
│   │   └── api_steps.py
│   │
│   └── environment.py             # Hooks & setup/teardown logic
│
├── support/                       # Framework support files
│   ├── pages/                     # Page Object Model (POM) for UI
│   │   ├── base_page.py
│   │   ├── login_page.py
│   │   └── profile_page.py
│   │
│   ├── api/                       # API client classes
│   │   ├── base_api.py
│   │   └── user_api.py
│   │
│   ├── locators/                  # Web element locators
│   │   └── login_locators.py
│   │
│   └── utils/                     # Utility functions
│       ├── config_reader.py
│       └── logger.py
│
├── config/                        # Configuration files
│   ├── behave.ini                 # Behave configuration
│   └── settings.json              # Test environment settings
│
├── drivers/                       # WebDriver executables
│   └── chromedriver.exe
│
├── reports/                       # Test reports (auto-generated)
│   └── allure-results/
│
├── screenshots/                   # Screenshots on failure
│
├── requirements.txt               # Python dependencies
└── README.md
```

---

## Installation & Setup

### 1. Install Required Packages

Create a `requirements.txt` file:

```txt
behave==1.2.6
selenium==4.15.2
requests==2.31.0
PyHamcrest==2.0.4
allure-behave==2.13.2
python-dotenv==1.0.0
```

Install dependencies:

```bash
pip install -r requirements.txt
```

### 2. Configure Behave

Create `config/behave.ini`:

```ini
[behave]
# Output format
format = pretty
color = true
show_skipped = false
show_timings = true

# Paths
paths = features/

# Reporting
junit = true
junit_directory = reports/junit

# Logging
logging_level = INFO
logging_format = %(levelname)s:%(name)s:%(message)s

# User data (custom configuration)
[behave.userdata]
browser = chrome
headless = false
base_url = http://localhost:3000
api_base_url = http://localhost:8000/api
timeout = 10
screenshot_on_failure = true
```

### 3. Environment Settings

Create `config/settings.json`:

```json
{
  "environments": {
    "local": {
      "web_url": "http://localhost:3000",
      "api_url": "http://localhost:8000/api"
    },
    "staging": {
      "web_url": "https://staging.example.com",
      "api_url": "https://api-staging.example.com"
    },
    "production": {
      "web_url": "https://example.com",
      "api_url": "https://api.example.com"
    }
  },
  "test_users": {
    "valid_user": {
      "username": "testuser@example.com",
      "password": "Test@123"
    },
    "admin_user": {
      "username": "admin@example.com",
      "password": "Admin@123"
    }
  }
}
```

---

## Part 1: Web UI Testing with Selenium

### Step 1: Create Page Object Model (POM)

**Why POM?** Separates web elements from test logic, making tests easier to maintain.

#### Base Page Class

Create `support/pages/base_page.py`:

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException, NoSuchElementException

class BasePage:
    """Base class for all page objects"""
    
    def __init__(self, driver):
        self.driver = driver
        self.timeout = 10
    
    def find_element(self, locator):
        """Find a single element with explicit wait"""
        try:
            element = WebDriverWait(self.driver, self.timeout).until(
                EC.presence_of_element_located(locator)
            )
            return element
        except TimeoutException:
            raise Exception(f"Element {locator} not found within {self.timeout} seconds")
    
    def find_elements(self, locator):
        """Find multiple elements"""
        return self.driver.find_elements(*locator)
    
    def click(self, locator):
        """Click on an element"""
        element = WebDriverWait(self.driver, self.timeout).until(
            EC.element_to_be_clickable(locator)
        )
        element.click()
    
    def enter_text(self, locator, text):
        """Enter text into an input field"""
        element = self.find_element(locator)
        element.clear()
        element.send_keys(text)
    
    def get_text(self, locator):
        """Get text from an element"""
        element = self.find_element(locator)
        return element.text
    
    def is_visible(self, locator):
        """Check if element is visible"""
        try:
            element = WebDriverWait(self.driver, self.timeout).until(
                EC.visibility_of_element_located(locator)
            )
            return True
        except TimeoutException:
            return False
    
    def take_screenshot(self, filename):
        """Take screenshot"""
        self.driver.save_screenshot(f"screenshots/{filename}.png")
```

#### Locators

Create `support/locators/login_locators.py`:

```python
from selenium.webdriver.common.by import By

class LoginPageLocators:
    """All locators for Login Page"""
    
    # Input fields
    USERNAME_INPUT = (By.ID, "username")
    PASSWORD_INPUT = (By.ID, "password")
    
    # Buttons
    LOGIN_BUTTON = (By.CSS_SELECTOR, "button[type='submit']")
    FORGOT_PASSWORD_LINK = (By.LINK_TEXT, "Forgot Password?")
    
    # Messages
    ERROR_MESSAGE = (By.CSS_SELECTOR, ".error-message")
    SUCCESS_MESSAGE = (By.CSS_SELECTOR, ".success-message")
    
    # Navigation
    LOGO = (By.CSS_SELECTOR, ".app-logo")
```

#### Page Class

Create `support/pages/login_page.py`:

```python
from support.pages.base_page import BasePage
from support.locators.login_locators import LoginPageLocators

class LoginPage(BasePage):
    """Page Object for Login Page"""
    
    def __init__(self, driver):
        super().__init__(driver)
        self.locators = LoginPageLocators
    
    def navigate_to_login(self, base_url):
        """Navigate to login page"""
        self.driver.get(f"{base_url}/login")
    
    def enter_username(self, username):
        """Enter username"""
        self.enter_text(self.locators.USERNAME_INPUT, username)
    
    def enter_password(self, password):
        """Enter password"""
        self.enter_text(self.locators.PASSWORD_INPUT, password)
    
    def click_login_button(self):
        """Click login button"""
        self.click(self.locators.LOGIN_BUTTON)
    
    def login(self, username, password):
        """Complete login action"""
        self.enter_username(username)
        self.enter_password(password)
        self.click_login_button()
    
    def get_error_message(self):
        """Get error message text"""
        return self.get_text(self.locators.ERROR_MESSAGE)
    
    def is_login_successful(self):
        """Check if login was successful by checking URL"""
        return "/dashboard" in self.driver.current_url
```

### Step 2: Write Feature File

Create `features/web/login.feature`:

```gherkin
Feature: User Login
  As a registered user
  I want to log into the application
  So that I can access my account

  Background:
    Given I navigate to the login page

  @web @smoke @positive
  Scenario: Successful login with valid credentials
    When I enter username "testuser@example.com"
    And I enter password "Test@123"
    And I click the login button
    Then I should be redirected to the dashboard
    And I should see a welcome message

  @web @negative
  Scenario: Login with invalid credentials
    When I enter username "invalid@example.com"
    And I enter password "WrongPass123"
    And I click the login button
    Then I should see an error message "Invalid username or password"
    And I should remain on the login page

  @web @negative
  Scenario Outline: Login with missing credentials
    When I enter username "<username>"
    And I enter password "<password>"
    And I click the login button
    Then I should see an error message "<error_message>"

    Examples:
      | username              | password  | error_message                |
      |                       | Test@123  | Username is required         |
      | testuser@example.com  |           | Password is required         |
      |                       |           | Username and Password required|
```

### Step 3: Implement Step Definitions

Create `features/steps/web_steps.py`:

```python
from behave import given, when, then
from hamcrest import assert_that, equal_to, contains_string, is_
from support.pages.login_page import LoginPage

@given('I navigate to the login page')
def step_navigate_to_login(context):
    """Navigate to login page"""
    context.login_page = LoginPage(context.driver)
    context.login_page.navigate_to_login(context.base_url)

@when('I enter username "{username}"')
def step_enter_username(context, username):
    """Enter username"""
    context.login_page.enter_username(username)

@when('I enter password "{password}"')
def step_enter_password(context, password):
    """Enter password"""
    context.login_page.enter_password(password)

@when('I click the login button')
def step_click_login(context):
    """Click login button"""
    context.login_page.click_login_button()

@then('I should be redirected to the dashboard')
def step_check_dashboard_redirect(context):
    """Verify redirect to dashboard"""
    assert_that(context.login_page.is_login_successful(), is_(True),
                "User should be redirected to dashboard")

@then('I should see a welcome message')
def step_verify_welcome_message(context):
    """Verify welcome message appears"""
    # Implementation depends on your app structure
    pass

@then('I should see an error message "{expected_message}"')
def step_verify_error_message(context, expected_message):
    """Verify error message"""
    actual_message = context.login_page.get_error_message()
    assert_that(actual_message, contains_string(expected_message),
                f"Expected error: {expected_message}, but got: {actual_message}")

@then('I should remain on the login page')
def step_verify_still_on_login(context):
    """Verify still on login page"""
    assert_that("/login" in context.driver.current_url, is_(True),
                "Should remain on login page")
```

---

## Part 2: REST API Testing

### Step 1: Create API Base Class

Create `support/api/base_api.py`:

```python
import requests
import json

class BaseAPI:
    """Base class for all API interactions"""
    
    def __init__(self, base_url):
        self.base_url = base_url
        self.session = requests.Session()
        self.headers = {
            'Content-Type': 'application/json',
            'Accept': 'application/json'
        }
    
    def set_auth_token(self, token):
        """Set authentication token"""
        self.headers['Authorization'] = f'Bearer {token}'
    
    def get(self, endpoint, params=None):
        """GET request"""
        url = f"{self.base_url}{endpoint}"
        response = self.session.get(url, headers=self.headers, params=params)
        return response
    
    def post(self, endpoint, payload=None):
        """POST request"""
        url = f"{self.base_url}{endpoint}"
        response = self.session.post(url, headers=self.headers, 
                                     data=json.dumps(payload))
        return response
    
    def put(self, endpoint, payload=None):
        """PUT request"""
        url = f"{self.base_url}{endpoint}"
        response = self.session.put(url, headers=self.headers, 
                                    data=json.dumps(payload))
        return response
    
    def delete(self, endpoint):
        """DELETE request"""
        url = f"{self.base_url}{endpoint}"
        response = self.session.delete(url, headers=self.headers)
        return response
    
    def patch(self, endpoint, payload=None):
        """PATCH request"""
        url = f"{self.base_url}{endpoint}"
        response = self.session.patch(url, headers=self.headers, 
                                      data=json.dumps(payload))
        return response
```

### Step 2: Create API Client

Create `support/api/user_api.py`:

```python
from support.api.base_api import BaseAPI

class UserAPI(BaseAPI):
    """API client for User endpoints"""
    
    def __init__(self, base_url):
        super().__init__(base_url)
        self.users_endpoint = "/users"
    
    def create_user(self, user_data):
        """Create a new user"""
        return self.post(self.users_endpoint, user_data)
    
    def get_user(self, user_id):
        """Get user by ID"""
        return self.get(f"{self.users_endpoint}/{user_id}")
    
    def get_all_users(self):
        """Get all users"""
        return self.get(self.users_endpoint)
    
    def update_user(self, user_id, user_data):
        """Update user"""
        return self.put(f"{self.users_endpoint}/{user_id}", user_data)
    
    def delete_user(self, user_id):
        """Delete user"""
        return self.delete(f"{self.users_endpoint}/{user_id}")
    
    def login_user(self, credentials):
        """Login user"""
        return self.post("/auth/login", credentials)
```

### Step 3: Write API Feature File

Create `features/api/user_api.feature`:

```gherkin
Feature: User Management API
  As an API consumer
  I want to manage users via REST API
  So that I can perform CRUD operations

  @api @smoke
  Scenario: Create a new user
    Given I have valid user data
      """
      {
        "name": "John Doe",
        "email": "john.doe@example.com",
        "password": "SecurePass123",
        "role": "user"
      }
      """
    When I send a POST request to "/users"
    Then the response status code should be 201
    And the response should contain user details
    And the response should have a field "id"
    And the response field "email" should be "john.doe@example.com"

  @api @smoke
  Scenario: Get user by ID
    Given a user exists with ID "123"
    When I send a GET request to "/users/123"
    Then the response status code should be 200
    And the response should contain user details

  @api @negative
  Scenario: Create user with missing required fields
    Given I have invalid user data
      """
      {
        "name": "Jane Doe"
      }
      """
    When I send a POST request to "/users"
    Then the response status code should be 400
    And the response should contain error details
    And the error message should contain "email is required"

  @api @crud
  Scenario: Complete user CRUD operations
    # Create
    Given I have valid user data for user "testuser"
    When I create a new user
    Then the response status code should be 201
    And I save the user ID as "created_user_id"
    
    # Read
    When I get the user with ID "{created_user_id}"
    Then the response status code should be 200
    
    # Update
    When I update the user with ID "{created_user_id}" with new email "updated@example.com"
    Then the response status code should be 200
    And the response field "email" should be "updated@example.com"
    
    # Delete
    When I delete the user with ID "{created_user_id}"
    Then the response status code should be 204
    
    # Verify deletion
    When I get the user with ID "{created_user_id}"
    Then the response status code should be 404

  @api @data-driven
  Scenario Outline: Validate user creation with different inputs
    Given I have user data with name "<name>" and email "<email>"
    When I send a POST request to "/users"
    Then the response status code should be <status_code>
    And the response should match expected result "<result>"

    Examples:
      | name       | email                | status_code | result  |
      | John Doe   | john@example.com     | 201         | success |
      | Jane Doe   | jane@example.com     | 201         | success |
      |            | noemail@example.com  | 400         | error   |
      | No Email   |                      | 400         | error   |
      | Test User  | invalid-email        | 400         | error   |
```

### Step 4: Implement API Step Definitions

Create `features/steps/api_steps.py`:

```python
from behave import given, when, then
from hamcrest import assert_that, equal_to, has_key, contains_string, is_in
import json

@given('I have valid user data')
def step_prepare_valid_user_data(context):
    """Prepare valid user data from docstring"""
    context.payload = json.loads(context.text)

@given('I have invalid user data')
def step_prepare_invalid_user_data(context):
    """Prepare invalid user data"""
    context.payload = json.loads(context.text)

@given('a user exists with ID "{user_id}"')
def step_user_exists(context, user_id):
    """Assume user exists (for testing purposes)"""
    context.user_id = user_id

@given('I have valid user data for user "{username}"')
def step_prepare_user_data_with_username(context, username):
    """Prepare user data with specific username"""
    context.payload = {
        "name": username,
        "email": f"{username}@example.com",
        "password": "Test@123"
    }

@when('I send a POST request to "{endpoint}"')
def step_send_post_request(context, endpoint):
    """Send POST request"""
    context.response = context.api_client.post(endpoint, context.payload)

@when('I send a GET request to "{endpoint}"')
def step_send_get_request(context, endpoint):
    """Send GET request"""
    context.response = context.api_client.get(endpoint)

@when('I create a new user')
def step_create_user(context):
    """Create a new user"""
    context.response = context.user_api.create_user(context.payload)

@when('I get the user with ID "{user_id}"')
def step_get_user(context, user_id):
    """Get user by ID"""
    # Replace placeholder if exists
    if user_id.startswith('{') and user_id.endswith('}'):
        user_id = context.saved_values.get(user_id.strip('{}'))
    context.response = context.user_api.get_user(user_id)

@when('I update the user with ID "{user_id}" with new email "{email}"')
def step_update_user(context, user_id, email):
    """Update user email"""
    if user_id.startswith('{') and user_id.endswith('}'):
        user_id = context.saved_values.get(user_id.strip('{}'))
    update_data = {"email": email}
    context.response = context.user_api.update_user(user_id, update_data)

@when('I delete the user with ID "{user_id}"')
def step_delete_user(context, user_id):
    """Delete user"""
    if user_id.startswith('{') and user_id.endswith('}'):
        user_id = context.saved_values.get(user_id.strip('{}'))
    context.response = context.user_api.delete_user(user_id)

@when('I have user data with name "{name}" and email "{email}"')
def step_prepare_user_data_params(context, name, email):
    """Prepare user data from parameters"""
    context.payload = {
        "name": name if name else None,
        "email": email if email else None
    }

@then('the response status code should be {status_code:d}')
def step_verify_status_code(context, status_code):
    """Verify response status code"""
    assert_that(context.response.status_code, equal_to(status_code),
                f"Expected {status_code}, but got {context.response.status_code}")

@then('the response should contain user details')
def step_verify_user_details(context):
    """Verify response contains user details"""
    response_data = context.response.json()
    assert_that(response_data, has_key('name'))
    assert_that(response_data, has_key('email'))

@then('the response should have a field "{field_name}"')
def step_verify_field_exists(context, field_name):
    """Verify field exists in response"""
    response_data = context.response.json()
    assert_that(response_data, has_key(field_name),
                f"Field '{field_name}' not found in response")

@then('the response field "{field_name}" should be "{expected_value}"')
def step_verify_field_value(context, field_name, expected_value):
    """Verify field value in response"""
    response_data = context.response.json()
    actual_value = response_data.get(field_name)
    assert_that(actual_value, equal_to(expected_value),
                f"Expected {field_name}={expected_value}, but got {actual_value}")

@then('I save the user ID as "{variable_name}"')
def step_save_user_id(context, variable_name):
    """Save user ID from response"""
    if not hasattr(context, 'saved_values'):
        context.saved_values = {}
    response_data = context.response.json()
    context.saved_values[variable_name] = response_data.get('id')

@then('the response should contain error details')
def step_verify_error_details(context):
    """Verify error details in response"""
    response_data = context.response.json()
    assert_that(response_data, has_key('error'),
                "Response should contain error details")

@then('the error message should contain "{text}"')
def step_verify_error_message(context, text):
    """Verify error message contains specific text"""
    response_data = context.response.json()
    error_message = response_data.get('error', '')
    assert_that(text.lower() in error_message.lower(), 
                f"Expected error message to contain '{text}', but got: {error_message}")

@then('the response should match expected result "{result}"')
def step_verify_result(context, result):
    """Verify result matches expectation"""
    if result == "success":
        assert_that(context.response.status_code, is_in([200, 201]))
    elif result == "error":
        assert_that(context.response.status_code, is_in([400, 404, 500]))
```

---

## Part 3: Environment Setup (Hooks & Context)

### Environment File

Create `features/environment.py`:

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from support.api.base_api import BaseAPI
from support.api.user_api import UserAPI
import json
import os

def before_all(context):
    """
    Runs once before all tests
    Setup global configurations
    """
    # Setup logging
    context.config.setup_logging()
    
    # Load test settings
    with open('config/settings.json', 'r') as f:
        context.settings = json.load(f)
    
    # Get environment from config or default to 'local'
    env = context.config.userdata.get('env', 'local')
    context.base_url = context.settings['environments'][env]['web_url']
    context.api_base_url = context.settings['environments'][env]['api_url']
    
    # Get test configuration
    context.browser_name = context.config.userdata.get('browser', 'chrome')
    context.headless = context.config.userdata.getbool('headless', False)
    context.timeout = int(context.config.userdata.get('timeout', 10))
    
    print(f"\n{'='*50}")
    print(f"Test Environment: {env}")
    print(f"Web URL: {context.base_url}")
    print(f"API URL: {context.api_base_url}")
    print(f"Browser: {context.browser_name}")
    print(f"Headless: {context.headless}")
    print(f"{'='*50}\n")


def before_feature(context, feature):
    """
    Runs before each feature
    """
    print(f"\nStarting Feature: {feature.name}")


def before_scenario(context, scenario):
    """
    Runs before each scenario
    Setup browser and API clients based on tags
    """
    # Check if scenario requires web browser
    if 'web' in scenario.effective_tags:
        # Setup browser
        context.driver = setup_browser(context)
        print(f"  Browser started for scenario: {scenario.name}")
    
    # Check if scenario requires API testing
    if 'api' in scenario.effective_tags:
        # Setup API client
        context.api_client = BaseAPI(context.api_base_url)
        context.user_api = UserAPI(context.api_base_url)
        context.saved_values = {}  # For storing values between steps
        print(f"  API client initialized for scenario: {scenario.name}")


def after_scenario(context, scenario):
    """
    Runs after each scenario
    Cleanup browser and take screenshot on failure
    """
    # Take screenshot on failure (if web test)
    if scenario.status == 'failed' and 'web' in scenario.effective_tags:
        if hasattr(context, 'driver'):
            screenshot_name = f"{scenario.name.replace(' ', '_')}_{scenario.status}"
            screenshot_path = f"screenshots/{screenshot_name}.png"
            
            # Create screenshots directory if it doesn't exist
            os.makedirs('screenshots', exist_ok=True)
            
            context.driver.save_screenshot(screenshot_path)
            print(f"  Screenshot saved: {screenshot_path}")
    
    # Quit browser
    if hasattr(context, 'driver'):
        context.driver.quit()
        print(f"  Browser closed after scenario: {scenario.name}")
    
    # Print scenario result
    status_symbol = "✓" if scenario.status == 'passed' else "✗"
    print(f"  {status_symbol} Scenario: {scenario.name} [{scenario.status.upper()}]")


def after_feature(context, feature):
    """
    Runs after each feature
    """
    print(f"\nCompleted Feature: {feature.name}")


def after_all(context):
    """
    Runs once after all tests
    Final cleanup
    """
    print(f"\n{'='*50}")
    print("All tests completed!")
    print(f"{'='*50}\n")


def setup_browser(context):
    """
    Setup and return WebDriver instance
    """
    browser_name = context.browser_name.lower()
    
    if browser_name == 'chrome':
        chrome_options = Options()
        
        if context.headless:
            chrome_options.add_argument('--headless')
        
        chrome_options.add_argument('--no-sandbox')
        chrome_options.add_argument('--disable-dev-shm-usage')
        chrome_options.add_argument('--disable-gpu')
        chrome_options.add_argument('--window-size=1920,1080')
        
        # For Windows, specify driver path
        # service = Service('drivers/chromedriver.exe')
        # driver = webdriver.Chrome(service=service, options=chrome_options)
        
        # If chromedriver is in PATH
        driver = webdriver.Chrome(options=chrome_options)
        
    elif browser_name == 'firefox':
        from selenium.webdriver.firefox.options import Options as FirefoxOptions
        firefox_options = FirefoxOptions()
        
        if context.headless:
            firefox_options.add_argument('--headless')
        
        driver = webdriver.Firefox(options=firefox_options)
    
    else:
        raise ValueError(f"Browser '{browser_name}' not supported")
    
    driver.maximize_window()
    driver.implicitly_wait(context.timeout)
    
    return driver
```

---

## Part 4: Utilities & Helper Functions

### Configuration Reader

Create `support/utils/config_reader.py`:

```python
import json
import os

class ConfigReader:
    """Read configuration from JSON file"""
    
    @staticmethod
    def load_config(config_file='config/settings.json'):
        """Load configuration from file"""
        if not os.path.exists(config_file):
            raise FileNotFoundError(f"Config file not found: {config_file}")
        
        with open(config_file, 'r') as f:
            return json.load(f)
    
    @staticmethod
    def get_test_user(config, user_type='valid_user'):
        """Get test user credentials"""
        users = config.get('test_users', {})
        return users.get(user_type)
    
    @staticmethod
    def get_environment_config(config, env='local'):
        """Get environment-specific configuration"""
        environments = config.get('environments', {})
        return environments.get(env)
```

### Custom Logger

Create `support/utils/logger.py`:

```python
import logging
import os
from datetime import datetime

class TestLogger:
    """Custom logger for test execution"""
    
    @staticmethod
    def setup_logger(name='behave_tests'):
        """Setup and return logger"""
        # Create logs directory
        os.makedirs('logs', exist_ok=True)
        
        # Create logger
        logger = logging.getLogger(name)
        logger.setLevel(logging.DEBUG)
        
        # Create file handler
        log_file = f"logs/test_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"
        file_handler = logging.FileHandler(log_file)
        file_handler.setLevel(logging.DEBUG)
        
        # Create console handler
        console_handler = logging.StreamHandler()
        console_handler.setLevel(logging.INFO)
        
        # Create formatter
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        file_handler.setFormatter(formatter)
        console_handler.setFormatter(formatter)
        
        # Add handlers
        logger.addHandler(file_handler)
        logger.addHandler(console_handler)
        
        return logger
```

---

## Part 5: Running Tests

### Basic Commands

```bash
# Run all tests
behave

# Run specific feature file
behave features/web/login.feature

# Run tests with specific tag
behave --tags=@smoke

# Run tests excluding a tag
behave --tags=~@wip

# Run tests with multiple tags (AND)
behave --tags=@web --tags=@smoke

# Run tests with multiple tags (OR)
behave --tags=@web,@api

# Run in headless mode
behave -D headless=true

# Run with specific browser
behave -D browser=firefox

# Run on specific environment
behave -D env=staging

# Run with verbose output
behave -v

# Run and stop on first failure
behave --stop

# Generate Allure report
behave -f allure_behave.formatter:AllureFormatter -o reports/allure-results
allure serve reports/allure-results
```

### Run Script

Create `run_tests.sh`:

```bash
#!/bin/bash

# Full Stack Behave Test Execution Script

echo "================================"
echo "Starting Full Stack Tests"
echo "================================"

# Set environment variables
export PYTHONPATH=.

# Create necessary directories
mkdir -p reports/allure-results
mkdir -p screenshots
mkdir -p logs

# Run tests and generate Allure report
behave \
  -f allure_behave.formatter:AllureFormatter \
  -o reports/allure-results \
  --format=pretty \
  --no-capture \
  --tags=@smoke \
  -D headless=false \
  -D env=local

# Generate and open Allure report
echo ""
echo "Generating Allure Report..."
allure generate reports/allure-results -o reports/allure-html --clean
allure open reports/allure-html

echo ""
echo "================================"
echo "Test Execution Completed!"
echo "================================"
```

Make it executable:

```bash
chmod +x run_tests.sh
./run_tests.sh
```

---

## Part 6: Best Practices

### 1. **Use PyHamcrest for Assertions**

PyHamcrest provides better error messages than standard assertions:

```python
from hamcrest import assert_that, equal_to, contains_string, is_, has_key

# Instead of:
assert context.status_code == 200

# Use:
assert_that(context.status_code, equal_to(200), "Status code should be 200")
```

### 2. **Tag Your Scenarios**

Use tags to organize and run specific test subsets:

```gherkin
@smoke @web @positive
Scenario: User can login successfully

@api @regression @negative
Scenario: API returns 404 for non-existent user
```

### 3. **Use Scenario Outlines for Data-Driven Testing**

```gherkin
Scenario Outline: Validate multiple user inputs
  Given I have user with name "<name>" and email "<email>"
  When I create the user
  Then the response code should be <status>

  Examples:
    | name  | email           | status |
    | John  | john@test.com   | 201    |
    | Jane  | jane@test.com   | 201    |
    |       | invalid@test.com| 400    |
```

### 4. **Keep Step Definitions Simple**

Step definitions should delegate logic to Page Objects or API clients:

```python
# Good
@when('I login with valid credentials')
def step_login(context):
    context.login_page.login(username, password)

# Avoid
@when('I login with valid credentials')
def step_login(context):
    driver.find_element(By.ID, "username").send_keys(username)
    driver.find_element(By.ID, "password").send_keys(password)
    driver.find_element(By.ID, "submit").click()
```

### 5. **Use Context to Share State**

```python
# Save data in context
context.user_id = response.json()['id']

# Use it in later steps
context.api.get_user(context.user_id)
```

### 6. **Create Reusable Base Classes**

Both BasePage and BaseAPI provide common functionality that specific pages/APIs inherit.

### 7. **Handle Test Data Properly**

- Use fixtures or environment hooks for test data setup
- Clean up test data after execution
- Use unique identifiers to avoid conflicts

### 8. **Screenshot on Failure**

Already implemented in `environment.py` `after_scenario` hook.

---

## Part 7: Advanced Topics

### Parallel Execution

Install `behave-parallel`:

```bash
pip install behave-parallel
```

Run tests in parallel:

```bash
behave --processes 4 --parallel-element scenario
```

### CI/CD Integration

Example GitHub Actions workflow (`.github/workflows/tests.yml`):

```yaml
name: Behave Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        
    - name: Install Chrome
      run: |
        wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
        sudo apt install ./google-chrome-stable_current_amd64.deb
    
    - name: Run tests
      run: |
        behave -f allure_behave.formatter:AllureFormatter -o allure-results
    
    - name: Generate Allure Report
      if: always()
      run: |
        allure generate allure-results -o allure-report
    
    - name: Upload Allure Report
      if: always()
      uses: actions/upload-artifact@v2
      with:
        name: allure-report
        path: allure-report
```

### Database Testing

Add database steps for full integration testing:

```python
# features/steps/database_steps.py
import psycopg2

@given('the database is clean')
def step_clean_database(context):
    """Clean test database"""
    conn = psycopg2.connect(context.db_connection_string)
    cursor = conn.cursor()
    cursor.execute("DELETE FROM users WHERE email LIKE '%test%'")
    conn.commit()
    conn.close()

@then('the user should be in the database')
def step_verify_user_in_db(context):
    """Verify user exists in database"""
    conn = psycopg2.connect(context.db_connection_string)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE email = %s", 
                   (context.user_email,))
    result = cursor.fetchone()
    assert_that(result, is_not(None), "User should exist in database")
    conn.close()
```

---

## Summary

This guide covered:

1. ✅ **Project structure** - Organized, scalable framework
2. ✅ **Web UI testing** - Using Selenium + Page Object Model
3. ✅ **API testing** - REST API testing with requests library
4. ✅ **Environment setup** - Hooks, context management, browser setup
5. ✅ **Step definitions** - Clean, reusable step implementations
6. ✅ **Best practices** - Industry-standard patterns
7. ✅ **Utilities** - Configuration, logging, screenshots
8. ✅ **Running tests** - Multiple execution options
9. ✅ **Advanced topics** - CI/CD, parallel execution, database testing

### Key Takeaways

- **Separation of Concerns**: Page Objects, API clients, and step definitions are separate
- **Reusability**: Base classes provide common functionality
- **Readability**: Gherkin syntax makes tests readable for everyone
- **Maintainability**: Changes in UI/API only affect specific page/API classes
- **Flexibility**: Easy to add new tests, pages, and API endpoints
- **CI/CD Ready**: Designed for automated execution in pipelines

### Next Steps

1. Clone or create the project structure
2. Install dependencies
3. Write your first feature file
4. Implement page objects and step definitions
5. Run tests locally
6. Integrate with CI/CD
7. Generate and review reports

---

**Happy Testing! 🚀**
