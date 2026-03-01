# XBlock Architecture Pattern

Every XBlock in this project follows the same structure. This document is the reference implementation.

---

## File Structure

```
my-xblock/
├── setup.py                          ← Package metadata + entry point registration
├── setup.cfg                         ← Optional build config
├── tox.ini                           ← Test runner config
├── MANIFEST.in                       ← Includes static files in package
├── my_xblock/
│   ├── __init__.py                   ← Exposes MyXBlock class
│   ├── my_xblock.py                  ← Main XBlock class (all logic here)
│   ├── static/
│   │   ├── html/
│   │   │   ├── student.html          ← Student view template
│   │   │   └── studio.html           ← Studio edit view template
│   │   ├── js/
│   │   │   ├── student.js            ← Student-facing JavaScript
│   │   │   └── studio.js             ← Studio JavaScript
│   │   └── css/
│   │       └── style.css             ← XBlock styles
│   └── tests/
│       ├── __init__.py
│       └── test_xblock.py            ← pytest tests (target: 80%+ coverage)
└── README.md
```

---

## setup.py

```python
from setuptools import setup

setup(
    name='my-xblock',
    version='0.1.0',
    description='My custom XBlock for ASU Global Connect',
    packages=['my_xblock'],
    install_requires=[
        'XBlock',
    ],
    entry_points={
        'xblock.v1': [
            'my_xblock = my_xblock.my_xblock:MyXBlock',
        #   ^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        #   name used    Python path to XBlock class
        #   in Studio
        ],
    },
    package_data={
        'my_xblock': [
            'static/css/*.css',
            'static/html/*.html',
            'static/js/*.js',
        ],
    },
)
```

---

## my_xblock.py — Full Reference Implementation

```python
import pkg_resources
import logging

from xblock.core import XBlock
from xblock.fields import Integer, String, Dict, Float, Boolean, Scope
from xblock.fragment import Fragment

log = logging.getLogger(__name__)


class MyXBlock(XBlock):
    """
    My custom XBlock.
    """

    # =========================================================================
    # FIELDS
    # =========================================================================
    # Fields define what data this XBlock stores and who can see it.

    # --- Settings (instructor-configurable, shared across all students) ---
    display_name = String(
        display_name="Display Name",
        default="My XBlock",
        scope=Scope.settings,
        help="The name shown to students and in Studio"
    )
    max_attempts = Integer(
        display_name="Max Attempts",
        default=3,
        scope=Scope.settings,
        help="How many times a student can attempt this activity"
    )

    # --- Content (author-defined course content) ---
    question_text = String(
        default="What is your answer?",
        scope=Scope.content,
        help="The question shown to students"
    )

    # --- User State (per-student, per-XBlock-instance data) ---
    student_answer = String(
        default="",
        scope=Scope.user_state,
        help="The student's submitted answer"
    )
    attempts = Integer(
        default=0,
        scope=Scope.user_state,
        help="Number of attempts this student has made"
    )
    completed = Boolean(
        default=False,
        scope=Scope.user_state,
        help="Whether the student has completed this activity"
    )
    score = Float(
        default=0.0,
        scope=Scope.user_state,
        help="The student's score (0.0 to 1.0)"
    )

    # =========================================================================
    # VIEWS
    # =========================================================================

    def student_view(self, context=None):
        """
        The primary view rendered in the LMS for students.
        Must return a Fragment containing HTML, CSS, JS.
        """
        html = self.resource_string("static/html/student.html")
        frag = Fragment(html.format(self=self))
        frag.add_css(self.resource_string("static/css/style.css"))
        frag.add_javascript(self.resource_string("static/js/student.js"))

        # Pass data to the JavaScript initializer
        frag.initialize_js('MyXBlock', {
            'attempts': self.attempts,
            'max_attempts': self.max_attempts,
            'completed': self.completed,
            'score': self.score,
        })
        return frag

    def studio_view(self, context=None):
        """
        Instructor-facing edit view in Studio.
        """
        html = self.resource_string("static/html/studio.html")
        frag = Fragment(html.format(self=self))
        frag.add_javascript(self.resource_string("static/js/studio.js"))
        frag.initialize_js('MyXBlockStudio')
        return frag

    # =========================================================================
    # HANDLERS (called from JavaScript via AJAX)
    # =========================================================================

    @XBlock.json_handler
    def submit_answer(self, data, suffix=''):
        """
        Called when a student submits an answer.
        data: parsed JSON dict from the JS POST body
        Returns: JSON-serializable dict
        """
        if self.completed:
            return {'error': 'Already completed'}

        if self.attempts >= self.max_attempts:
            return {'error': 'No attempts remaining'}

        answer = data.get('answer', '')
        self.student_answer = answer
        self.attempts += 1

        # Score the answer
        is_correct = self._check_answer(answer)
        if is_correct:
            self.score = 1.0
            self.completed = True
            xp = 100

            # Publish XP event to Gamification XBlock
            self.runtime.publish(self, 'xp_earned', {
                'xp': xp,
                'action': 'my_xblock_completed',
                'user_id': self.runtime.user_id,
            })

            # Report grade to gradebook
            self.runtime.publish(self, 'grade', {
                'value': self.score,
                'max_value': 1.0,
            })
        else:
            self.score = 0.0

        log.info(
            "User %s submitted answer. Correct: %s. Attempts: %d",
            self.runtime.user_id, is_correct, self.attempts
        )

        return {
            'success': True,
            'correct': is_correct,
            'attempts': self.attempts,
            'attempts_remaining': self.max_attempts - self.attempts,
            'score': self.score,
        }

    @XBlock.json_handler
    def save_settings(self, data, suffix=''):
        """
        Called from Studio to save instructor settings.
        """
        self.display_name = data.get('display_name', self.display_name)
        self.max_attempts = data.get('max_attempts', self.max_attempts)
        self.question_text = data.get('question_text', self.question_text)
        return {'success': True}

    # =========================================================================
    # HELPERS
    # =========================================================================

    def _check_answer(self, answer):
        """Internal: check if student's answer is correct."""
        # Implement actual grading logic here
        return len(answer) > 0  # placeholder

    def resource_string(self, path):
        """Load a resource file relative to this XBlock's package."""
        data = pkg_resources.resource_string(__name__, path)
        return data.decode("utf8")

    @staticmethod
    def workbench_scenarios():
        """Scenarios for the XBlock Workbench (local testing tool)."""
        return [
            ("MyXBlock default", "<my_xblock/>"),
            ("MyXBlock with settings", '<my_xblock max_attempts="5"/>'),
        ]
```

---

## static/html/student.html

```html
<div class="my-xblock" id="my-xblock-{self.scope_ids.usage_id}">
    <h3>{self.display_name}</h3>
    <p class="question">{self.question_text}</p>

    <div class="answer-area">
        <input type="text" class="answer-input" placeholder="Your answer..." />
        <button class="submit-btn">Submit</button>
    </div>

    <div class="feedback" style="display:none;">
        <p class="feedback-message"></p>
    </div>

    <div class="progress">
        Attempts: <span class="attempts-count">{self.attempts}</span> / {self.max_attempts}
    </div>
</div>
```

---

## static/js/student.js

```javascript
function MyXBlock(runtime, element, initArgs) {
    // initArgs = data passed from frag.initialize_js()
    var attempts = initArgs.attempts;
    var maxAttempts = initArgs.max_attempts;
    var completed = initArgs.completed;

    var submitUrl = runtime.handlerUrl(element, 'submit_answer');

    // Bind submit button
    $(element).find('.submit-btn').click(function() {
        if (completed || attempts >= maxAttempts) {
            return;
        }

        var answer = $(element).find('.answer-input').val();

        $.ajax({
            type: 'POST',
            url: submitUrl,
            data: JSON.stringify({ answer: answer }),
            contentType: 'application/json',
            success: function(response) {
                attempts = response.attempts;
                $(element).find('.attempts-count').text(attempts);
                $(element).find('.feedback').show();

                if (response.correct) {
                    $(element).find('.feedback-message')
                        .text('Correct! +100 XP')
                        .addClass('correct');
                    completed = true;
                    $(element).find('.submit-btn').prop('disabled', true);
                } else {
                    $(element).find('.feedback-message')
                        .text('Incorrect. ' + response.attempts_remaining + ' attempts left.')
                        .addClass('incorrect');
                }
            },
            error: function(xhr) {
                console.error('Handler error:', xhr.responseText);
            }
        });
    });
}
```

---

## tests/test_xblock.py

```python
"""Tests for MyXBlock."""
import unittest
from unittest.mock import MagicMock, patch
from xblock.test.toy_runtime import ToyRuntime


class TestMyXBlock(unittest.TestCase):

    def setUp(self):
        """Set up test fixtures."""
        from my_xblock.my_xblock import MyXBlock

        # Create XBlock with mock runtime
        self.runtime = ToyRuntime()
        self.block = MyXBlock(self.runtime, scope_ids=MagicMock())
        self.block.runtime.user_id = 42

    def test_student_view_renders(self):
        """student_view returns a Fragment with HTML."""
        frag = self.block.student_view({})
        self.assertIsNotNone(frag)
        self.assertIn('my-xblock', frag.content)

    def test_submit_answer_increments_attempts(self):
        """Submitting an answer increments attempt count."""
        self.assertEqual(self.block.attempts, 0)
        self.block.submit_answer({'answer': 'test'}, suffix='')
        self.assertEqual(self.block.attempts, 1)

    def test_cannot_exceed_max_attempts(self):
        """Returns error when max attempts exceeded."""
        self.block.max_attempts = 2
        self.block.attempts = 2
        result = self.block.submit_answer({'answer': 'test'}, suffix='')
        self.assertIn('error', result)

    def test_completed_publishes_xp_event(self):
        """Completing the activity publishes an xp_earned event."""
        self.block.runtime.publish = MagicMock()
        self.block.submit_answer({'answer': 'something'}, suffix='')
        self.block.runtime.publish.assert_called_with(
            self.block, 'xp_earned', unittest.mock.ANY
        )

    def test_save_settings_updates_fields(self):
        """save_settings handler updates XBlock fields."""
        result = self.block.save_settings({
            'display_name': 'New Name',
            'max_attempts': 5,
        }, suffix='')
        self.assertEqual(result['success'], True)
        self.assertEqual(self.block.display_name, 'New Name')
        self.assertEqual(self.block.max_attempts, 5)


if __name__ == '__main__':
    unittest.main()
```

---

## Key Rules

1. **Never raw SQL** — use Django ORM or XBlock field storage
2. **Scope correctly** — student data = `Scope.user_state`, config = `Scope.settings`
3. **Always return a dict** from handlers — it gets JSON-serialized
4. **Always publish XP events** when a student completes an activity
5. **80%+ test coverage** — non-negotiable
6. **CSRF in JS** — include `X-CSRFToken` header in all AJAX requests
