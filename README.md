# Streamlining Test Reporting: Selective Control of Screenshots and Video Recordings in Automation Frameworks

Capturing screenshots and video recordings during test execution is essential for debugging and improving test reporting. However, not all test cases require these resources. For instance, tests for highly dynamic components or minor validations may not benefit from screenshots or recordings.

In this blog, we’ll explore how to use a unified configuration file (`config.json`) to selectively enable or disable screenshots and video recordings for specific test cases across three frameworks: **Robot Framework, Pytest, and WebdriverIO**

## **1. A Unified Configuration File `config.json`**

The central piece of this approach is the `config.json` file, which allows us to define:

- Test cases where screenshots should be disabled.
- Test cases where video recording should be disabled.

**Sample `config.json` :**

```json
{
  "recordVideo": true,
  "disable_screenshots": [
    "Verify user is able to filter product by brand",
    "Verify user can submit a bug report"
  ],
  "disable_screen_recording": [
    "Verify pop-up window is displayed",
    "Verify download link works correctly"
  ]
}
```

- `recordVideo` : Global flag to enable/disable video recording.
- `disable_screenshots` : List of test case names where screenshots should be skipped.
- `disable_screen_recording` : List of test case names where video recording should be skipped.

---

## **2. Implementation in Robot Framework (Python)**

### **Selective Disabling of Screenshots and Video Recordings in Robot Framework**

In Robot Framework, we use a custom listener to capture screenshots and decide whether to skip them based on the `config.json` file.

**Code for Screenshot Automation in Robot Framework :**

```python
import json
import os
from robot.libraries.BuiltIn import BuiltIn
from robot.api import logger
import re
import AllureLibrary

class EndUserKeywordScreenshot:
    ROBOT_LISTENER_API_VERSION = 3

    def __init__(self):
        # Load configuration
        self.config = self.load_config()
        self.disable_screenshots = set(self.config.get("disable_screenshots", []))

    def load_config(self):
        """Load configuration from a JSON file."""
        config_path = os.path.join(os.getcwd(), 'tests/Config/config.json')
        try:
            with open(config_path, 'r') as file:
                return json.load(file)
        except FileNotFoundError:
            logger.warn(f"Could not load config.json: File not found at {config_path}")
        except Exception as e:
            logger.warn(f"Error loading config.json: {e}")
        return {}

    def sanitize_filename(self, name):
        """Sanitize a string to make it safe for filenames."""
        return re.sub(r'[<>:"/\\|?*]', '_', name)

    def end_user_keyword(self, data, implementation, result):
        """Executed after each user keyword ends."""
        try:
            # Get the name of the currently running test case
            test_name = BuiltIn().get_variable_value("${TEST NAME}")

            # Check if screenshots are disabled for this test case
            if test_name in self.disable_screenshots:
                logger.info(f"Skipping screenshot for test case: {test_name}")
                return  # Skip taking screenshot

            # Capture screenshot if not disabled
            output_dir = os.path.join(os.getcwd(), 'tests/Artifacts/screenshots')
            if not os.path.exists(output_dir):
                os.makedirs(output_dir)

            sanitized_data = self.sanitize_filename(str(data))
            screenshot_path = os.path.join(output_dir, f'screenshot_{sanitized_data}.png')
            self.browserLibrary = BuiltIn().get_library_instance('Browser')
            self.browserLibrary.take_screenshot(filename=screenshot_path)
            try:
                AllureLibrary.attach_file(source=screenshot_path, name='screenshot', attachment_type='PNG')
            except Exception as e:
                print(f"Failed to attach screenshot: {e}")
        except Exception as e:
            logger.warn(f"Failed to take screenshot: {e}")
```

### **Selective Disabling of Video Recordings**

```python
class EndUserKeywordVideoRecording:
    def __init__(self):
        self.config = self.load_config()
        self.disable_video = set(self.config.get("disable_screen_recording", []))

    def start_video_recording(self, test_name):
        """Check if video recording should be enabled."""
        if test_name in self.disable_video:
            print(f"Skipping video recording for: {test_name}")
            return False  # Skip video recording
        return True  # Enable video recording
```

---

## **2. Implementation in Pytest Framework (Python)**

### **Selective Disabling of Screenshots**

In Pytest, we use fixtures and the `config.json` file to control screenshots. Screenshots are skipped for tests listed in `disable_screenshots`.

**Code for Screenshot Automation in Pytest :**

```python
import pytest
import json
from allure_commons.types import AttachmentType
import allure

# Load configuration from JSON file
def load_config():
    with open('config.json', 'r') as f:
        return json.load(f)

config = load_config()

@pytest.fixture(autouse=True)
def capture_screenshot_on_failure(page, request):
    # Get the test name from the request object
    test_name = request.node.name
    disable_screenshots = config.get('disable_screenshots', [])

    # Check if screenshots are disabled for this test case
    if test_name in disable_screenshots:
        yield
        return  # Skip screenshot capture

    # Capture screenshot if the test fails
    yield
    if request.node.rep_call.failed:
        screenshot_path = f'screenshots/{test_name}.png'
        page.screenshot(path=screenshot_path)
        allure.attach.file(screenshot_path, name="Screenshot", attachment_type=AttachmentType.PNG)
```

### **Selective Disabling of Video Recordings**

```python
@pytest.fixture(scope="function")
def context_with_video(browser):
    # Load config to check if video recording is enabled or disabled
    disable_video = config.get("disable_screen_recording", [])
    test_name = pytest.current_test.name

    if test_name in disable_video:
        # Skip video recording for this test case
        yield browser.new_context()
        return

    # Enable video recording for the test
    context = browser.new_context(record_video_dir="videos/")
    yield context
    context.close()  # Ensure the video is saved when the context is closed
```

---

## **3. Implementation in WebdriverIO Framework (JavaScript)**

### **Selective Disabling of Screenshots**

WebdriverIO's `afterTest` hook is used to capture screenshots. The `config.json` file is checked to skip screenshots for specific tests.

```javascript
const fs = require("fs");
const allure = require("allure-commandline");
const config = require("./config.json");

exports.config = {
  afterTest: async function (
    test,
    context,
    { error, result, duration, passed }
  ) {
    const disabledScreenshots = config.disable_screenshots || [];
    if (disabledScreenshots.includes(test.title)) {
      console.log(`Skipping screenshot for: ${test.title}`);
      return;
    }

    // Take screenshot if the test fails
    if (!passed) {
      const screenshotPath = `./screenshots/${test.title}.png`;
      await browser.saveScreenshot(screenshotPath);
      allure.addAttachment(
        "Screenshot",
        Buffer.from(fs.readFileSync(screenshotPath)),
        "image/png"
      );
    }
  },
};
```

### **Selective Disabling of Video Recordings**

In WebdriverIO, we control video recording using the `afterSuite` hook.

```javascript
const Video = require("wdio-video-reporter");
const config = require("./config.json");

exports.config = {
  reporters: [
    [
      Video,
      {
        saveAllVideos: false, // Only save videos for failed tests
        videoSlowdownMultiplier: 3, // Slows video for better visibility
      },
    ],
  ],

  afterSuite: function (suite) {
    const config = require("./config.json");
    const disabledVideoRecording = config.disable_screen_recording || [];

    // Skip video recording for this suite
    if (disabledVideoRecording.includes(suite.name)) {
      console.log(`Skipping video recording for suite: ${suite.name}`);
    }
  },
};
```

---

## **Conclusion**

By integrating a single configuration file (`config.json`) across multiple frameworks, we’ve achieved a flexible and centralized mechanism for controlling screenshots and video recordings. This approach allows for selective disabling based on test case requirements, ensuring that unnecessary resources are not captured.

The unified `config.json` makes it easy to update or modify rules, enabling seamless customization across **Robot Framework, Pytest, and WebdriverIO**. This centralized approach improves maintainability and reduces resource usage during test execution.
