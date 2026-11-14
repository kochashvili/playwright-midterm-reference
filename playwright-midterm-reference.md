# Playwright + pytest midterm reference

**Windows PowerShell · PyCharm · Python sync API · Lectures 1–6 and the midterm**

Search with **Ctrl+F** for the action or API: `checkbox`, `get_by_role`, `parametrize`, `conftest.py`, `dropdown`, `alert`, `iframe`, `expect_popup`, `codegen`, `report.html`, or `ZIP`. Copy one example, then change the **URL, locator, input data, and expected result** to match your prompt. Follow the exam's rules for reference materials.

[Environment Setup](#environment-setup) · [Midterm Patterns](#midterm-patterns) · [Locators](#locators) · [Assertions and Waits](#assertions-and-waits) · [UI Actions](#ui-actions) · [Dialogs, Tabs, and Frames](#dialogs-tabs-and-frames) · [Browser Tools](#browser-tools) · [Page Objects](#page-objects) · [Excel and Submission](#excel-and-submission) · [Course Sources](#course-sources)

## Environment Setup

### 1. Create or open the project in PyCharm

Open **New Project**, choose a writable location such as `C:\Users\student\PycharmProjects\midterm`, and open the project. If PyCharm offers a **uv** interpreter, you can select it. If you create a plain project, the terminal steps below will create the uv environment. If the exam gives you a starter folder containing `pyproject.toml`, open that folder as the project.

Open PyCharm's **Terminal** tool window and use **PowerShell**. Run these to verify you're at the project root.

```powershell
Get-Location
Get-ChildItem -Force
uv --version
```

### 2. Find or install uv on Windows

If `uv --version` works, go to step 3. If it fails on the classroom PC, try the path from the professor's sample. This change applies to the current terminal.

```powershell
$env:Path += ";$env:APPDATA\Python\Scripts"
uv --version
```

If uv is missing and you are allowed to install it, use one installation method. Open a fresh terminal afterward. If PyCharm still has the older PATH, restart PyCharm.

```powershell
winget install --id astral-sh.uv -e
```

If WinGet is unavailable and the Python launcher works, this is another installation option. The second command finds the scripts folder for the Python installation that `py` runs and adds it to this PowerShell session.

```powershell
py -m pip install --user uv
$uvScripts = py -c "import sysconfig; print(sysconfig.get_path('scripts', scheme='nt_user'))"
$env:Path += ";$uvScripts"
uv --version
```

[Professor's Windows setup](https://github.com/nikajordania/lecture-2-playwright/blob/main/SETUP-UV.md) · [Official uv installation options](https://docs.astral.sh/uv/getting-started/installation/)

### 3. Set up the Python environment and Chromium

**Fresh project.** The professor's worked midterm uses Python 3.14. Use the version specified on your exam if it differs. Run `uv init` only when the project has no `pyproject.toml`. If PyCharm created one, open it and check its `requires-python` before choosing the interpreter. If you need 3.14 and the project's requirements allow it, run `uv python pin 3.14` in place of the `uv init` line.

```powershell
uv python install 3.14
uv init --bare --python 3.14
uv add pytest pytest-playwright pytest-html
uv run playwright install chromium
New-Item -ItemType Directory -Path tests -Force
uv run python --version
uv run pytest --version
```

**Starter project with dependencies already listed.** Run these from that project folder.

```powershell
uv sync
uv run playwright install chromium
```

uv creates `.venv` and records resolved dependencies in `uv.lock`. Running through `uv run` lets the terminal use that environment without an activation step. For PyCharm's editor and Run button, press **Ctrl+Alt+S**, search for **Python Interpreter**, and choose **Add Interpreter → Add Local Interpreter**. Select the existing uv environment, or browse to `.venv\Scripts\python.exe` as an existing interpreter. If two environments appear, use the interpreter inside the exam project. Search Settings for **Python Integrated Tools** and choose **pytest** as the default test runner if you want to use the gutter Run button. Use the terminal commands below when you need specific pytest flags.

[Midterm Python version and dependencies](https://github.com/nikajordania/software-testing-midterm-example/blob/main/pyproject.toml) · [uv project setup](https://docs.astral.sh/uv/concepts/projects/init/) · [Playwright installation](https://playwright.dev/python/docs/intro)

### 4. Write and run your first test

Create `tests/test_smoke.py` in PyCharm. pytest finds files and functions whose names start with `test_`. The plugin supplies `page` (a fresh browser tab for each test). The colon starts the function body and each line inside starts with four spaces.

```python
from playwright.sync_api import expect


def test_smoke(page):
    page.set_content("<h1>Ready</h1>")
    heading = page.get_by_role("heading", name="Ready")
    expect(heading).to_be_visible()
```

That smoke test runs with a local HTML string, so it checks your Python setup and browser without a practice website. Run it with:

```powershell
uv run pytest tests/test_smoke.py -v --headed
```

### Run one test, rerun failures, and debug

```powershell
uv run pytest -v
uv run pytest tests/test_smoke.py::test_smoke -v
uv run pytest -k "cart or checkbox" -v
uv run pytest --lf -v
uv run pytest -x --headed --slowmo 400
uv run pytest --collect-only -q
uv run pytest --tracing retain-on-failure --screenshot only-on-failure
```

- `--headed` shows the browser, `--slowmo 400` slows actions, `-x` stops after the first failure, and `--collect-only` shows what pytest found.
- **Fixture 'page' not found** → run `uv add pytest-playwright` in the project and execute pytest through `uv run`.
- **Executable doesn't exist** → run `uv run playwright install chromium`.
- **Unrecognized `--html` argument** → run `uv add pytest-html`.
- **Strict mode violation** → your locator matched multiple elements. Narrow it with `exact=True`, a parent locator, or `.filter(...)`.
- **Timeout** → read the failed action or assertion, check the locator and page, and try headed mode. For elements inside frames use `frame_locator`. For a click that opens a second tab, capture the new page first.

[pytest plugin and CLI flags](https://playwright.dev/python/docs/test-runners) · [Professor's exam commands](https://github.com/nikajordania/software-testing-midterm-example/blob/main/commands.txt)

## Midterm Patterns

### Checkbox — check, uncheck, assert initial and final state

```python
from playwright.sync_api import expect


def test_checkbox_states(page):
    page.goto("https://practice.expandtesting.com/checkboxes")
    first = page.locator("#checkbox1")
    second = page.locator("#checkbox2")
    expect(first).not_to_be_checked()
    expect(second).to_be_checked()

    first.check()
    second.uncheck()
    expect(first).to_be_checked()
    expect(second).not_to_be_checked()
```

**Change** the URL, two IDs, and states expected at each step. `check()` sets a box to checked. `uncheck()` clears it. `click()` toggles it. [Midterm Task 1](https://github.com/nikajordania/software-testing-midterm-example/blob/main/tests/test_task1_checkboxes.py)

### Keyboard — press a key and check its result with parametrize

```python
import pytest
from playwright.sync_api import expect


@pytest.mark.parametrize("key, expected", [
    ("Escape", "You entered: ESCAPE"),
    ("a", "You entered: A"),
])
def test_key_result(page, key, expected):
    page.goto("https://practice.expandtesting.com/key-presses")
    page.locator("#target").press(key)
    expect(page.locator("#result")).to_have_text(expected)
```

**Change** the input locator, result locator, and each (input, expected output) pair. pytest runs this twice. Names in the decorator must match function parameters. [Midterm Task 2](https://github.com/nikajordania/software-testing-midterm-example/blob/main/tests/test_task2_key_presses.py)

### Login fixture — share setup through conftest.py

Save this in **`tests/conftest.py`**. pytest finds it automatically. `return page` gives the logged-in tab to tests that name the `inventory` fixture.

```python
import pytest
from playwright.sync_api import expect


@pytest.fixture
def inventory(page):
    page.goto("https://www.saucedemo.com/")
    page.get_by_placeholder("Username").fill("standard_user")
    page.get_by_placeholder("Password").fill("secret_sauce")
    page.locator("#login-button").click()
    expect(page.locator("[data-test='title']")).to_have_text("Products")
    return page
```

**Change** the demo URL, fields, test-account data, and successful-login check. The sample prompt asks for a fixture even though the published Task 3 solution puts login inside the test. This version follows the prompt. These are public SauceDemo test credentials. [Sample prompt](https://drive.google.com/file/d/1CJGbLtaP-50munlFPbTajGckxZpShnuS/view) · [Professor's fixture example](https://github.com/nikajordania/software-testing-midterm-example/blob/main/tests/conftest.py)

### Cart badge — loop, filter a product card, and check the count

Save in `tests/test_cart.py` with the fixture above in `tests/conftest.py`. Every row below gives pytest one list of products and one expected count.

```python
import pytest
from playwright.sync_api import expect


@pytest.mark.parametrize("products, count", [
    (["Sauce Labs Backpack"], 1),
    (["Sauce Labs Backpack", "Sauce Labs Bike Light"], 2),
    (["Sauce Labs Backpack", "Sauce Labs Bike Light", "Sauce Labs Onesie"], 3),
])
def test_cart_badge(inventory, products, count):
    cards = inventory.locator("[data-test='inventory-item']")
    for product in products:
        card = cards.filter(has_text=product)
        card.get_by_role("button", name="Add to cart", exact=True).click()

    badge = inventory.locator("[data-test='shopping-cart-badge']")
    expect(badge).to_have_text(str(count))
```

**Change** the card locator, names, button name, and badge locator. Scoping the button to `card` tells Playwright which of the repeated Add to cart buttons to click. `str(count)` turns the number into the text shown on screen. [Midterm Task 3](https://github.com/nikajordania/software-testing-midterm-example/blob/main/tests/test_task3_cart_badge.py)

### Fixture cleanup — yield

`yield` hands the object to the test. The code in `finally` runs during cleanup, including when the test fails. Useful when you create a context yourself.

```python
import pytest


@pytest.fixture
def small_page(browser):
    context = browser.new_context(viewport={"width": 800, "height": 600})
    try:
        yield context.new_page()
    finally:
        context.close()


def test_window_size(small_page):
    assert small_page.viewport_size == {"width": 800, "height": 600}
```

[Lecture fixture examples](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/data)

## Locators

### Role, label, text, placeholder, alt, title, and test ID

Paste the line you need **inside** a test after `page.goto(...)`. The strings are examples to replace. A locator describes how to find an element. Calling `.click()`, `.fill()`, or `expect(...)` uses that locator against the current page.

```python
button = page.get_by_role("button", name="Submit", exact=True)
link = page.get_by_role("link", name="Cart")
field = page.get_by_label("Email")
username = page.get_by_placeholder("Username")
message = page.get_by_text("Order placed", exact=True)
logo = page.get_by_alt_text("Store logo")
help_icon = page.get_by_title("Help")
save = page.get_by_test_id("save-button")
```

`get_by_test_id` defaults to `data-testid`. For sites using `data-test`, CSS works directly with `page.locator("[data-test='save-button']")`.

### CSS, dynamic IDs, XPath, and locator chaining

```python
by_id = page.locator("#userName")
by_class = page.locator(".inventory_item")
by_attribute = page.locator("input[type='email']")
starts_with = page.locator("[id^='item-']")
row = page.get_by_role("row").filter(has_text="Samsung galaxy s6")
delete_button = row.get_by_role("link", name="Delete")
second_row = page.get_by_role("row").nth(1)  # Index 1 is the second match
output_name = page.locator("#output").locator("#name")
cell = page.locator("xpath=//td[normalize-space()='Smith']/following-sibling::td[1]")
```

**Change** selector strings to match the DOM. `#` means ID, `.` means class, `[key='value']` matches an attribute, and a space means “inside.” Use role/name or a stable attribute when an ID changes on reload. Use `.first`, `.last`, or `.nth()` when position is part of the scenario. For a unique row or card, filter by its content first. [Lecture locator exercises](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/locators) · [Professor's locator notes](https://github.com/nikajordania/lecture-3-playwright/blob/main/playwright-locators.md)

## Assertions and Waits

### Pick the assertion for the expected result

Copy only the assertion for your scenario. Replace selectors, expected values, and the sample URL pattern.

```python
import re
from playwright.sync_api import expect

expect(page.locator("#message")).to_have_text("Saved")
expect(page.locator("#message")).to_contain_text("Saved")
expect(page.locator("#email")).to_have_value("student@example.com")
expect(page.locator("#save")).to_be_enabled()
expect(page.locator("#save")).to_be_disabled()
expect(page.locator("#spinner")).to_be_hidden()
expect(page.locator("#result")).to_be_visible()
expect(page.locator("#result")).to_be_attached()
expect(page.locator(".cart-item")).to_have_count(2)
expect(page.locator(".cart-item")).to_have_count(0)
expect(page.locator("table tbody tr td:first-child")).to_have_text(["One", "Two"])
expect(page.locator("#password")).to_have_attribute("type", "password")
expect(page.locator("#email")).to_contain_class("field-error")
expect(page).to_have_title(re.compile("Products"))
expect(page).to_have_url(re.compile(r"/inventory\.html$"))
```

`to_have_text` checks the full text. `to_contain_text` checks for a substring. `to_have_value` checks what an input contains. `to_be_hidden` also succeeds when the element is absent. Use `to_have_count(0)` when removal from the DOM is the expected result. To require two conditions, write two assertions. To wait for either a success notification or an error during debugging, use `expect(success.or_(error).first).to_be_visible()`, then assert the specific outcome your test requires. [Lecture assertions and combinations](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/check)

### Wait for a result instead of sleeping

```python
from playwright.sync_api import expect


def test_delayed_result(page):
    page.goto("https://the-internet.herokuapp.com/dynamic_loading/2")
    page.get_by_role("button", name="Start").click()
    expect(page.locator("#finish")).to_contain_text("Hello World!", timeout=10_000)
```

Playwright actions wait for the relevant readiness conditions. `expect(...)` retries until its assertion passes or times out. Plain `assert`, `.is_visible()`, `.count()`, and `.all()` are snapshots. Wait for the list to reach the expected state before iterating over `.all()`. For an explicit wait use `locator.wait_for(state="visible")` or `page.wait_for_url("**/inventory.html")`. [Lecture wait examples](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/waits)

## UI Actions

### Fill and submit a form — positive and negative cases

```python
from playwright.sync_api import expect


def test_form_outputs_name(page):
    page.goto("https://demoqa.com/text-box")
    page.locator("#userName").fill("Anna Test")
    page.locator("#userEmail").fill("anna@example.com")
    page.get_by_role("button", name="Submit", exact=True).click()
    output = page.locator("#output")
    expect(output.locator("#name")).to_have_text("Name:Anna Test")
    expect(output.locator("#email")).to_have_text("Email:anna@example.com")


def test_invalid_email(page):
    page.goto("https://demoqa.com/text-box")
    page.locator("#userName").fill("Anna Test")
    email = page.locator("#userEmail")
    email.fill("invalid.email")
    page.locator("#submit").click()
    expect(email).to_contain_class("field-error")
    expect(page.locator("#output #name")).to_have_count(0)
```

**Change** fields, input data, and what proves success or rejection. `fill()` replaces the value. `clear()` empties it. If only one field is required, leave the optional fields untouched and verify exactly what appears. For boundary cases, parametrize inputs just below, at, and just above the specified limit. [Lecture 2 manual-to-automated cases](https://github.com/nikajordania/lecture-2-playwright/blob/main/test_demoqa_text_box.py) · [Lecture 1 form and login scenarios](https://github.com/nikajordania/test-automation-project-example/tree/main/tests)

### Click, double click, and right click

```python
from playwright.sync_api import expect


def test_mouse_buttons(page):
    page.goto("https://demoqa.com/buttons")
    page.get_by_role("button", name="Double Click Me", exact=True).dblclick()
    expect(page.locator("#doubleClickMessage")).to_contain_text("double click")
    page.get_by_role("button", name="Right Click Me", exact=True).click(button="right")
    expect(page.locator("#rightClickMessage")).to_contain_text("right click")
```

For a regular left click use `.click()`. Change the button names and result selectors. [Lecture actions](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/action)

### Dropdown — native select versus custom component

```python
from playwright.sync_api import expect


def test_native_dropdown(page):
    page.goto("https://the-internet.herokuapp.com/dropdown")
    dropdown = page.locator("#dropdown")
    dropdown.select_option(label="Option 2")
    expect(dropdown).to_have_value("2")
```

Use `select_option(value="2")` if you know the value attribute. For a custom dropdown, click the control, then click the visible option with `get_by_role("option", name="Option 2", exact=True)` if it has that role. Then check the control displays your selection. [Lecture dropdown](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_4/test_03_dropdown.py)

### Hover and drag and drop

```python
from playwright.sync_api import expect


def test_hover_reveals_caption(page):
    page.goto("https://the-internet.herokuapp.com/hovers")
    card = page.locator(".figure").nth(1)
    caption = card.locator(".figcaption")
    expect(caption).to_be_hidden()
    card.hover()
    expect(caption).to_contain_text("name: user2")
    expect(caption).to_be_visible()


def test_drag_into_target(page):
    page.goto("https://demoqa.com/droppable")
    target = page.locator("#simpleDropContainer #droppable")
    page.locator("#draggable").drag_to(target)
    expect(target).to_have_text("Dropped!")
```

**Change** the source, destination or hover target, and the result. [Lecture 4 UI actions](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_4)

### Keyboard focus, shortcuts, Enter, and slider arrows

```python
from playwright.sync_api import expect


def test_tab_moves_focus(page):
    page.goto("https://the-internet.herokuapp.com/login")
    page.get_by_label("Username").press("Tab")
    expect(page.get_by_label("Password")).to_be_focused()


def test_slider_arrows(page):
    page.goto("https://the-internet.herokuapp.com/horizontal_slider")
    slider = page.locator("input[type='range']")
    for _ in range(3):
        slider.press("ArrowRight")
    expect(page.locator("#range")).to_have_text("1.5")
```

On a field, `.press("Enter")` may submit the form. `.press("ControlOrMeta+A")` followed by `.press("Delete")` selects and clears the text. Use `.press("Escape")` for Escape. [Lecture 5 keyboard](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_5/test_04_keyboard.py)

### Upload a file or download a file

```python
from playwright.sync_api import expect


def test_upload(page, tmp_path):
    sample = tmp_path / "notes.txt"
    sample.write_text("Example upload", encoding="utf-8")
    page.goto("https://the-internet.herokuapp.com/upload")
    page.locator("#file-upload").set_input_files(sample)
    page.locator("#file-submit").click()
    expect(page.locator("#uploaded-files")).to_have_text("notes.txt")


def test_download(page, tmp_path):
    page.goto("https://the-internet.herokuapp.com/download")
    link = page.locator(".example a").first
    expected_name = link.inner_text().strip()
    with page.expect_download() as info:
        link.click()
    download = info.value
    assert download.suggested_filename == expected_name
    saved = tmp_path / download.suggested_filename
    download.save_as(saved)
    assert saved.stat().st_size > 0
```

`tmp_path` is a temporary directory pytest supplies. Start waiting for the download **before** clicking. For an upload with a real file, use an appropriate path, such as `tmp_path / "notes.txt"` or a raw Windows path `r"C:\Users\student\Documents\notes.txt"`. [Upload lecture](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_4/test_06_upload.py) · [Download lecture](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_5/test_02_download.py)

### Navigate, go back, go forward, and reload

```python
from playwright.sync_api import expect


def test_browser_history(page):
    home = "https://the-internet.herokuapp.com/"
    page.goto(home)
    page.get_by_role("link", name="Checkboxes", exact=True).click()
    expect(page).to_have_url(home + "checkboxes")
    page.go_back()
    expect(page).to_have_url(home)
    page.go_forward()
    page.reload()
    expect(page.get_by_role("heading", name="Checkboxes")).to_be_visible()
```

**Change** the starting URL, link, and evidence the reloaded page is ready. For persistence checks, change something before `reload()` and assert that the change survives. [Lecture navigation](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_4/test_08_back_forward_reload.py)

## Dialogs, Tabs, and Frames

### JavaScript alert, confirm, prompt, and exact dialog message

```python
from playwright.sync_api import expect


def test_confirm_cancel(page):
    page.goto("https://the-internet.herokuapp.com/javascript_alerts")
    seen = []

    def handle_dialog(dialog):
        seen.append((dialog.type, dialog.message))
        dialog.dismiss()

    page.once("dialog", handle_dialog)
    page.get_by_role("button", name="Click for JS Confirm").click()
    assert seen == [("confirm", "I am a JS Confirm")]
    expect(page.locator("#result")).to_have_text("You clicked: Cancel")
```

Register the listener before clicking. For OK, use `dialog.accept()`. For a prompt with typed text, use `dialog.accept("Anna")`. For an HTML modal that is part of the page, find its button with a normal locator. [Lecture dialogs](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_4/test_07_dialogs.py)

### New tab or window — expect_popup

```python
import re

import pytest
from playwright.sync_api import expect


@pytest.mark.parametrize("button_name", ["New Tab", "New Window"])
def test_popup(page, button_name):
    original = "https://demoqa.com/browser-windows"
    page.goto(original)
    with page.expect_popup() as popup_info:
        page.get_by_role("button", name=button_name, exact=True).click()

    popup = popup_info.value
    expect(popup).to_have_url(re.compile(r"/sample$"))
    expect(popup.locator("#sampleHeading")).to_have_text("This is a sample page")
    popup.close()
    expect(page).to_have_url(original)
```

**Change** the opening control and popup checks. Register before the click. Use `page.context.expect_page()` when catching a new page anywhere in that browser context. This is an optional task in the sample. [Midterm Task 4](https://github.com/nikajordania/software-testing-midterm-example/blob/main/tests/test_task4_new_tab_and_window.py)

### Iframe, nested frame, and Shadow DOM

```python
from playwright.sync_api import expect


def test_embedded_heading(page):
    page.goto("https://demoqa.com/frames")
    frame = page.frame_locator("#frame1")
    expect(frame.locator("#sampleHeading")).to_have_text("This is a sample page")


def test_shadow_button(page):
    page.goto("https://practice.expandtesting.com/shadowdom")
    inside = page.locator("#shadow-host").locator("#my-btn")
    expect(inside).to_contain_text("inside a Shadow DOM")
```

For nested frames, call `.frame_locator(...)` at each level. In an open shadow root, CSS and role locators work across the boundary. XPath stops at that boundary. **Change** the iframe or host selector and the inner locator. [Lecture 5 frames and shadow DOM](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_5)

### HTTP Basic Auth — credentials attached to a browser context

```python
from playwright.sync_api import expect


def test_demo_basic_auth(browser):
    context = browser.new_context(http_credentials={"username": "admin", "password": "admin"})
    try:
        page = context.new_page()
        page.goto("https://the-internet.herokuapp.com/basic_auth")
        expect(page.get_by_text("Congratulations!", exact=False)).to_be_visible()
    finally:
        context.close()
```

Use this for the browser's HTTP authentication challenge on the assigned test site. For an HTML login form, use `fill()` and `click()` as in the login fixture. [Lecture 5 Basic Auth](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_5/test_03_basic_auth.py)

## Browser Tools

### Screenshots, Inspector, Trace Viewer, and Codegen

```python
def test_capture_page(page, tmp_path):
    page.set_content("<h1>Screenshot sample</h1>")
    screenshot = tmp_path / "page.png"
    page.screenshot(path=screenshot, full_page=True)
    assert screenshot.is_file()
```

For an element screenshot use `page.locator("#result").screenshot(path=tmp_path / "result.png")`. For inspection, insert `page.pause()` before the failing line and run headed. For a trace, run with `--tracing retain-on-failure`, then replace the example path below with the actual trace path printed or saved by pytest.

```powershell
$env:PWDEBUG = "1"
uv run pytest tests/test_smoke.py --headed
Remove-Item Env:PWDEBUG

uv run pytest --tracing retain-on-failure
uv run playwright show-trace ".\test-results\your-test-name\trace.zip"

uv run playwright codegen --target python-pytest https://www.saucedemo.com/
uv run playwright codegen --target python-pytest --test-id-attribute data-test https://www.saucedemo.com/
```

Codegen records locators and actions. Review the generated draft, name the test for the scenario, and add the assertions from the prompt. [Lecture screenshots](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_4/test_11_screenshots.py) · [Lecture Codegen notes](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_6/CODEGEN.md) · [Lecture tracing notes](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_6/TRACE.md)

### Browser engines, channels, viewport, mobile, and emulation

```powershell
uv run playwright install firefox webkit
uv run pytest --browser chromium --browser firefox --browser webkit
uv run pytest --browser chromium --browser-channel chrome
uv run pytest --device "iPhone 13" --browser webkit
```

`--browser-channel chrome` uses installed Google Chrome. For an installed Edge use `msedge`. For viewport changes inside a test, call `page.set_viewport_size({"width": 390, "height": 844})` before checking responsive UI.

```python
from playwright.sync_api import expect


def test_mobile_menu(browser, playwright):
    device = playwright.devices["iPhone 13"].copy()
    device.pop("default_browser_type", None)
    context = browser.new_context(**device)
    try:
        page = context.new_page()
        page.goto("https://practice.expandtesting.com/")
        expect(page.get_by_role("button", name="Toggle navigation")).to_be_visible()
    finally:
        context.close()
```

A context represents an isolated browser session. Set options at creation, for example `browser.new_context(locale="ka-GE", timezone_id="Asia/Tbilisi", color_scheme="dark")`. For a location-based UI, use the context's `geolocation={"latitude": 41.7151, "longitude": 44.8271}` with `permissions=["geolocation"]`. To set shared defaults across plugin pages, override the `browser_context_args` fixture and return a dictionary that includes its original values. [Lecture 5 mobile](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_5/test_06_mobile.py) · [Lecture 6 browsers and emulation](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_6)

### Console errors and JavaScript evaluate

```python
def test_page_title_and_errors(page):
    errors = []
    page.on("pageerror", lambda error: errors.append(str(error)))
    page.set_content("<title>Example</title><h1>Ready</h1>")
    assert page.evaluate("document.title") == "Example"
    assert errors == []
```

Register listeners before the action that might produce the event. `page.on("console", handler)` captures console messages, and `message.type` tells you whether the console message is an error. Wait for the real end of the workflow before checking the collected list. `locator.evaluate("el => el.textContent")` is useful for inspecting a property. Use Playwright actions for user interactions so that checks for visibility and clickability still apply. [Lecture console and evaluate](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_6)

### Network response, offline, and mocked response

```python
from playwright.sync_api import expect


def test_books_response(page):
    with page.expect_response("**/BookStore/v1/Books") as response_info:
        page.goto("https://demoqa.com/books")
    response = response_info.value
    assert response.status == 200
    expect(page.locator("tbody tr")).to_have_count(len(response.json()["books"]))


def test_empty_books(page):
    page.route("**/BookStore/v1/Books", lambda route: route.fulfill(json={"books": []}))
    page.goto("https://demoqa.com/books")
    expect(page.locator("tbody tr")).to_have_count(0)
```

**Change** the endpoint pattern and expected data. Add routes before requests can fire. For an offline UI test, use `context.set_offline(True)` and restore with `context.set_offline(False)` in `finally`. To simulate a failed request, use `route.abort()` on the request you are testing. [Lecture network examples](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_6/test_02_network.py) · [Lecture routing](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_6/test_10_route.py)

### Cookies, local storage, and storage_state

This example uses the `inventory` fixture above. It saves the session created by the normal demo login, then starts an independent context from the saved state.

```python
from playwright.sync_api import expect


def test_resume_demo_login(inventory, browser, tmp_path):
    assert any(cookie["name"] == "session-username" for cookie in inventory.context.cookies())
    state = tmp_path / "state.json"
    inventory.context.storage_state(path=state)

    context = browser.new_context(storage_state=state)
    try:
        page = context.new_page()
        page.goto("https://www.saucedemo.com/inventory.html")
        expect(page.locator("[data-test='title']")).to_have_text("Products")
    finally:
        context.close()
```

`context.cookies()` reads the context's cookies. `context.clear_cookies()` clears them for a logout or expiration check. To inspect app data in the current page, use `page.evaluate("localStorage.getItem('cart-contents')")`. Keep saved session files out of Git and the exam ZIP. [Lecture cookies](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_6/test_07_cookies.py) · [Lecture storage state](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_6/test_08_storage_state.py)

## Page Objects

### Small Page Object Model (POM) example

A page object stores locators and gives repeated actions a name. This snippet can run as one file. When the task asks for page classes under `tests/pages/`, move the classes into those modules and import them from the test. Replace login data, locators, and expected message to fit your site.

```python
from playwright.sync_api import expect


class LoginPage:
    def __init__(self, page):
        self.page = page
        self.username = page.get_by_label("Username")
        self.password = page.get_by_label("Password")
        self.submit = page.get_by_role("button", name="Login")
        self.message = page.locator("#flash")

    def open(self):
        self.page.goto("https://the-internet.herokuapp.com/login")

    def sign_in(self, username, password):
        self.username.fill(username)
        self.password.fill(password)
        self.submit.click()


def test_login_page_object(page):
    login = LoginPage(page)
    login.open()
    login.sign_in("tomsmith", "SuperSecretPassword!")
    expect(login.message).to_contain_text("You logged into a secure area!")
```

The sample optional cart task uses `LoginPage`, `InventoryPage`, and `CartPage`. It adds three products, verifies the badge and cart, removes a specific product, continues shopping, then logs out. See the original classes for the multi-file version. [Midterm Page Objects](https://github.com/nikajordania/software-testing-midterm-example/tree/main/tests/pages) · [Lecture 4 Page Objects](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_4/pages)

### Checkout, negative login, sorting, and price assertions

Use the same pattern for a longer workflow. For checkout, assert success at the end and add checks at meaningful steps such as cart contents or totals. For invalid login, parametrize the input and exact error as in the keyboard test. For sorting, read the displayed prices after the sort and compare against a sorted copy. `Decimal` represents currency without floating-point rounding.

```python
from decimal import Decimal

# Use inside a test after the app has loaded and applied the requested sort.
price_texts = page.locator("[data-test='inventory-item-price']").all_text_contents()
prices = [Decimal(text.removeprefix("$").strip()) for text in price_texts]
assert prices
assert prices == sorted(prices)
```

**Change** the price selector, currency format, and sort direction. Use `reverse=True` for descending order. [Lecture 6 checkout and sorting](https://github.com/nikajordania/lecture-3-playwright/blob/main/tests/lecture_6/test_13_checkout_e2e.py) · [Lecture 1 login, cart, and checkout](https://github.com/nikajordania/test-automation-project-example/blob/main/tests/test_saucedemo.py)

### Allure reporting — Lecture 1 project

Use these from the professor's **existing Lecture 1 checkout**, which supplies the dependencies and `allure_tool.py`. The Allure CLI needs Java. For the midterm's `report.html`, use the separate pytest-html commands below.

```powershell
uv sync
uv run playwright install chromium
uv run pytest --alluredir=allure-results -n 0
uv run python allure_tool.py serve
```

The project attaches failure screenshots in `conftest.py`, supports a report with step names and attachments, and uses pytest-xdist for parallel runs. `-n 0` runs serially for debugging. [Lecture 1 project README](https://github.com/nikajordania/test-automation-project-example)

## Excel and Submission

### Manual scenarios — Excel

The sample asks for two manually executed scenarios. The example is a positive add-to-cart case and a negative empty-checkout form case. Write the descriptions in Georgian and preserve exact UI labels and messages. Fill these nine columns.

| Column | What to write |
| --- | --- |
| ID | `TC-01`, `TC-02` |
| Title | The behavior being checked |
| Type | Positive or Negative |
| Preconditions | State needed before starting, such as an empty cart or one product already added |
| Steps | 5–8 numbered steps, one action per step |
| Test data | Product names, input values, or which fields are empty |
| Expected result | What should happen, with exact text or numbers tied to the relevant steps |
| Actual result | What you personally observed when you ran it |
| Status | Pass if actual matches expected, otherwise Fail |

Save as `test-cases-სახელი-გვარი.xlsx`, replacing the placeholders with your name as directed. A negative test **passes** when the app correctly rejects the bad input. [Sample prompts](https://drive.google.com/file/d/1CJGbLtaP-50munlFPbTajGckxZpShnuS/view) · [Completed manual cases](https://drive.google.com/file/d/1T4DQuduPytx2jiTSD7ydt4xm1qVRDGgA/view)

### report.html, commands.txt, and ZIP from PowerShell

Run this in the project folder after the tests are ready.

```powershell
uv run pytest -v --html=report.html --self-contained-html
Invoke-Item .\report.html
```

Make `commands.txt` in PyCharm or with `notepad.exe commands.txt`. Record the actual setup and testing commands you used. To package the sample's required files, edit the name and confirm the Excel filename matches. Put your required tests and any fixtures or page modules they use inside `tests\`.

```powershell
$name = "first-last"
$files = @(
    "pyproject.toml",
    "uv.lock",
    "tests",
    "report.html",
    "commands.txt",
    "test-cases-${name}.xlsx"
)
Compress-Archive -LiteralPath $files -DestinationPath "${name}-midterm.zip" -Force
tar -tf "${name}-midterm.zip"
```

Check the ZIP listing and open the report before submitting. The sample asks you to leave out `.venv`. Its archive example lists the optional exercise files too. Include those if you completed them, and follow the actual exam ticket for the exact files and naming. [Sample instructions](https://drive.google.com/file/d/1CJGbLtaP-50munlFPbTajGckxZpShnuS/view) · [PowerShell Compress-Archive](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.archive/compress-archive)

## Course Sources

The following public repositories were cloned before this guide was written. Local paths are relative to the Codex workspace that contains this guide.

| Material | Local checkout | GitHub |
| --- | --- | --- |
| Lecture 1 project. Pytest suite, login, forms, cart, checkout, Allure | `work/repos/test-automation-project-example` | [test-automation-project-example](https://github.com/nikajordania/test-automation-project-example) |
| Lecture 2. Windows uv setup, manual spreadsheet, text-box tests | `work/repos/lecture-2-playwright` | [lecture-2-playwright](https://github.com/nikajordania/lecture-2-playwright) |
| Lectures 3–6. Locators, actions, waits, fixtures, browser interactions, debugging, full checkout | `work/repos/lecture-3-playwright` | [lecture-3-playwright](https://github.com/nikajordania/lecture-3-playwright) |
| Midterm. Checkbox, keys, cart count, optional popup and POM | `work/repos/software-testing-midterm-example` | [software-testing-midterm-example](https://github.com/nikajordania/software-testing-midterm-example) |

More examples inside the lecture repo are indexed in its [Lecture 3 README](https://github.com/nikajordania/lecture-3-playwright), [Lecture 4 README](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_4), [Lecture 5 README](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_5), and [Lecture 6 README](https://github.com/nikajordania/lecture-3-playwright/tree/main/tests/lecture_6). The sample also mentions a lecture ZIP from class. These are the Playwright repos found on the professor's public profile.

Official references for the lab setup and syntax. [PyCharm uv environment](https://www.jetbrains.com/help/pycharm/uv.html) · [Playwright Python](https://playwright.dev/python/docs/intro) · [pytest parametrization](https://docs.pytest.org/en/stable/how-to/parametrize.html)
