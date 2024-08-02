# How to add Chrome extension in Selene?


## An example how to remove ads in Chrome with [uBlock Origin extension](https://chromewebstore.google.com/detail/ublock-origin/cjpalhdlnbpafiamejdnhcphjbkeiagm) by installing `.crx` file

### Preliminary steps
1. Download and store extension. (Example: by using [crxextractor](https://crxextractor.com/))

### Example project structure
```
📦my-project
 ┣ 📂my_project
 ┃ ┣ 📜__init__.py
 ┃ ┗ 📂resources
 ┃   ┣ 📜__init__.py
 ┃   ┗ 📂chrome_extensions
 ┃     ┗ 📜uBlock-Origin.crx
 ┣ 📂tests
 ┃ ┣ 📜test_ublock.py
 ┃ ┗ 📜conftest.py
 ┣ 📜readme.md
 ┣ 📜.gitignore
 ┣ 📜poetry.lock
 ┗ 📜pyproject.toml
```

### Code description

#### Helper to build absolute Path to resources

`my_project/resources/__init__.py`

```python
from pathlib import Path

path = Path(__file__).resolve().parent
```


#### Install extension and navigate into it's settings

`tests/conftest.py`

```python
import pytest
from selene import browser, have, Element, be
from selene.core.locator import Locator
from selenium import webdriver

from my_project import resources


@pytest.fixture(autouse=True)
def browser_with_ublock():
    ublock_path = resources.path / 'chrome_extensions/uBlock-Origin.crx'
    # ublock_id a unique constant for uBlock Origin extension
    ublock_id = 'cjpalhdlnbpafiamejdnhcphjbkeiagm'
    options = webdriver.ChromeOptions()
    options.add_extension(ublock_path)
    browser.config.driver_options = options

    browser.open('chrome://extensions/')
    js = f'''return document.querySelector('body > extensions-manager')
    .shadowRoot.querySelector('#items-list')
    .shadowRoot.querySelector('#{ublock_id}')
    .shadowRoot.querySelector('#card')'''
    card = Element(
        Locator('ublock extension card', lambda: browser.execute_script(js)),
        browser.config,
    )
    # Specific behaviour for uBlock extension.
    # Initially it is enabled, then disabled, then enabled again.
    card.should(have.css_class('disabled')).should(have.css_class('enabled'))
    # You might want to increase timeout.
    # card.with_(timeout=browser.config.timeout*1.5).should(...).should(...)
    browser.open(f'chrome-extension://{ublock_id}/dashboard.html#about.html')
    browser.switch_to.frame(browser.element('iframe')())
    browser.element('#aboutNameVer').should(be.visible).should(
        have.text('uBlock Origin')
    )

    yield

```

#### Verify that uBlock extension rules are applied to site with ads
`tests/test_ublock.py` 

```python
from selene import browser, have


def test_verify_applied_rules():
    # Act
    browser.open('https://d3ward.github.io/toolz/adblock')
    browser.element('#dlg_changelog button').with_(click_by_js=True).click()

    # Assert
    browser.element('#adb_test_r').should(have.text('0 not blocked'))

```