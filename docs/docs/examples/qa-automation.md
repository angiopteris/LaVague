# QA Automation with LaVague and Pytest

**What if you could generate test cases for your entire web app simply from product specs ?**

See it in action processing one use case to generate an automated login test:

!["qa_automation_demo"](../../assets/qa_automation_demo.gif)

You can run this example directly with a [CLI script available here](https://github.com/lavague-ai/LaVague/blob/main/examples/qa-automation/). We provide a sample `.feature` file containing a single use case. You can try it on this demo url: https://www.saucedemo.com/

This documentation will dive deeper in the code of our example script to give you a better understanding of how LaVague can automate testing. 

We will parse a Gherkin `.feature` file, run the test case with our web agent and use what the agent learned to generate a reusable automated test case usable by `pytest-bdd` 


### Pre-requisites

**Note**: We use OpenAI's models, for the embedding, LLM and Vision model. You will need to set the OPENAI_API_KEY variable in your local environment with a valid API key for this example to work.

If you don't have an OpenAI API key, please [get one here](https://platform.openai.com/docs/quickstart/developer-quickstart).

## Installation

For this example, we will use the OpenAI API Client and LaVague.

```bash
pip install openai lavague
```
We will need to set our OpenAI Key. If you are running this as a Colab, you can provide it through Colab secrets (see the key icon on the left-hand side of the Colab notebook) named 'OPENAI_API_KEY' and then convert it to an environment variable with the same name.

## Generating automated tests: a simple example

### Starting from a Gherkin test case and an URL

Gherkin is a language used in behavior-driven development (BDD) to describe feature behavior. 

Our example uses a Gherkin `.feature` file that describes a test case for a login feature for [this site](https://www.saucedemo.com/)

```gherkin
Feature: Swag Labs Login

  Scenario: Locked out user should be unable to log in
    Given I am on the Swag Labs login page
    When I enter "locked_out_user" into the username field
    And I enter "secret_sauce" into the password field
    And I click the "Login" button
    Then I should see an error message saying "Sorry, this user has been locked out."
```

We provide the path to this file and the URL of the associated websites as inputs to our script. 
```bash
python qa_automation_example.py --url https://www.saucedemo.com --feature-file tests/test_swag_labs.feature
```

### Using LaVague to run the test case once

We first parse the provided feature file to extract our test cases

```python
feature_name, feature_file_name, test_case = parse_feature_file(file_path)
```

We then initialize the LaVague agent with it's default components.

```python
selenium_driver = SeleniumDriver(headless=False)
world_model = WorldModel()
action_engine = ActionEngine(selenium_driver)
agent = WebAgent(world_model, action_engine)
```

We simply run the test case with LaVague. We will use recording of what the agent understood to generate a replayable `pytest` file

```python
agent.get(url)
agent.run(f"Run this test case: {test_case}")
```


### Processing the Results

After running the test case we retrieve relevant nodes from the final state of the page by performing Retrieval-Augmented Generation. We ask the action_engine which nodes would be most relevant to generate the final assert statement.

This helps in understanding the changes made to the state of the web page as a result of running the test case.

```python
nodes = action_engine.navigation_engine.get_nodes(f"We have ran the test case, generate the final assert statement.")
```

We parse the logs recorded by the agent to retrieve the screenshot of the final state of the page. This screenshot will be used in our call to GPT4o to generate the pytest file. 

```python
logs = agent.logger.return_pandas()
last_screenshot_path = get_latest_screenshot_path(logs.iloc[-1]["screenshots_path"])
b64_img = pil_image_to_base64(last_screenshot_path)
```

We also retrieve all the selenium code generated by the agent when running the test case
```python
selenium_code = "\n".join(logs['code'].dropna())
```

### Generating Pytest Code

We use the OpenAI API to generate pytest code based on the state of the last page, the selenium code. 

You can look at the prompts we use the [script available here](https://github.com/lavague-ai/LaVague/blob/main/examples/qa-automation/).

```python
def generate_pytest_code(url, feature_name, test_case, selenium_code, nodes, b64_img):
    client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
    completion = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
                "role": "system", 
                "content": SYSTEM_PROMPT
            },
            {
                "role": "user", 
                "content": [
                    {"type": "text", "text": build__prompt(url, feature_name, test_case, selenium_code, nodes)},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64_img}"}}
                ]
            }
        ]
    )
    return completion.choices[0].message.content

code = generate_pytest_code(url, feature_file_name, test_case, selenium_code, nodes, b64_img)
```

We can then cleanup the generated code and save it to our `tests` directory
```python
code = code.replace("```python", "").replace("```", "").replace("```\n", "").strip()

with open(f"./tests/{feature_name}.py", "w") as f:
    f.write(code)
```

## The generated code

Here's what the test case code looks like after simply running the test case once, recording the results, and using those to generate a `pytest-bdd` file. You can try running `pytest -v tests` to execute this file. 

```python
import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from pytest_bdd import scenarios, given, when, then

# Constants
BASE_URL = 'https://www.saucedemo.com/'

# Scenarios
scenarios('test_swag_labs.feature')

# Fixtures
@pytest.fixture
def browser():
    driver = webdriver.Chrome()
    driver.implicitly_wait(10)
    driver.get(BASE_URL)
    yield driver
    driver.quit()

# Steps
@given('I am on the Swag Labs login page')
def i_am_on_the_swag_labs_login_page(browser):
    pass

@when('I enter "locked_out_user" into the username field')
def i_enter_locked_out_user_into_the_username_field(browser):
    username_field = browser.find_element(By.XPATH, "/html/body/div/div/div[2]/div[1]/div/div/form/div[1]/input")
    username_field.send_keys("locked_out_user")

@when('I enter "secret_sauce" into the password field')
def i_enter_secret_sauce_into_the_password_field(browser):
    password_field = browser.find_element(By.XPATH, "/html/body/div/div/div[2]/div[1]/div/div/form/div[2]/input")
    password_field.send_keys("secret_sauce")

@when('I click the "Login" button')
def i_click_the_login_button(browser):
    login_button = browser.find_element(By.XPATH, "/html/body/div/div/div[2]/div[1]/div/div/form/input")
    try:
        browser.execute_script("arguments[0].click();", login_button)
    except Exception as e:
        pytest.fail(f"Failed to click login button: {e}")

@then('I should see an error message saying "Sorry, this user has been locked out."')
def i_should_see_an_error_message(browser):
    try:
        error_message = browser.find_element(By.XPATH, "/html/body/div/div/div[2]/div[1]/div/div/form/div[3]/h3")
        assert "Sorry, this user has been locked out." in error_message.text
    except Exception as e:
        pytest.fail(f"Error message not displayed: {e}")
```

## Running our tests

You can try running the generated test by using `pytest-bdd` by doing 
```
pytest -v tests
```

## Limitations and next steps

Current limitations of this example:

- The generation is not perfect, some test cases can fail, mostly because of selectors. Check the generated files for mistakes. 
- This script currently handles one scenario per `.feature` file
- Generate code can sometimes contain extra markdown from the API response and will need to be better striped
- We only write one final assert statement based on the state of the final step of the test case

Going further:

- Handle multiple scenarios per feature file
- Handle other testing frameworks
- Integrate with existing source of tests
- Run tests directly from product specs on JIRA

## Need help ? Found a bug ?
Join our [Discord](https://discord.gg/invite/SDxn9KpqX9) to ask us any questions!
