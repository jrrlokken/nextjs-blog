---
title: "Testing Python with Selenium"
date: "2021-02-15"
---

When it came time to write tests for my Issue Tracking System
I did not have extensive experience with any particular testing tools.
I had already written unit tests for authentication and the basic
functionality of the app, but wanted to expand my knowledge of testing
and testing tools. I reached for Selenium.

The first step is to install the Python bindings for Selenium,
which is as easy as:

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

`$ pip install selenium`

</div>

from your Python virtual environment. If you run into issues, or
want to learn more about installing and configuring Selenium, you can
find further information [here](https://selenium-python.readthedocs.io/installation.html). The selenium package does not provide a testing
framework, rather test cases can be written using Python's unittest
module. You could also use [pytest](https://docs.pytest.org/en/6.2.x/index.html) and [nose](https://nose.readthedocs.io/en/latest/).

---

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

```python
# Import required things:
import unittest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By

options = Options()
options.add_argument('--headless')

# Inherit TestCase class and create a new class
class ITSLoginCreation(unittest.TestCase):

  # Initialize the web driver
  def setUp(self):
    self.driver = webdriver.Chrome(options=options)

  # Our tests. The first test performs a get on the application's
  # login page and logs in a user. It then tests the application's
  # primary functionality, creating a new issue. The test name says
  # it all.

  def test_login_and_create_issue(self):
    # Setup the driver
    driver = self.driver
    # Perform a get request on the login page
    driver.get('https://its.joshualokken.tech/login')
    # Selenium allows for selection of elements by id or css selector
    email = driver.find_element(By.ID, 'email')
    password = driver.find_element(By.ID, 'password')
    button = driver.find_element(By.CSS_SELECTOR, '#login-button')
    # Send_keys sends the included text to the appropriate fields
    email.send_keys('george@example.com')
    password.send_keys('password')
    # Click the login button
    button.click()

    # Wait 5 seconds, then click the New Issue navigation link
    driver.implicitly_wait(5)
    newItemNav = driver.find_element(By.ID, 'new-issue')
    newItemNav.click()

    # Wait 5 seconds, then create and submit a new issue
    driver.implicitly_wait(5)
    title = driver.find_element(By.ID, 'title')
    text = driver.find_element(By.ID, 'text')
    submit = driver.find_element(By.ID, 'new-issue-submit')
    title.send_keys('**TEST ISSUE**')
    text.send_keys('Test issue description')
    submit.click()
    assert "Issue submitted" in driver.page_source

  def tearDown(self):
    self.driver.close()

if __name__ == "__main__":
unittest.main()
```

</div>
