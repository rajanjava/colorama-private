# Issue Resolution: Prevent winterm.set_title call on non-Windows / when convert is disabled

## 1. Upstream Issue Reference
- **Repository:** `tartley/colorama`
- **Issue:** [#407](https://github.com/tartley/colorama/issues/407) - `winterm incorrectly called on Linux when strip == True`
- **Related Issue:** [#392](https://github.com/tartley/colorama/issues/392) - `call_win32 function contains incomparable type (erase_line is not a known attribute of None)`
- **Status:** Open upstream

## 2. Problem Description
When running Colorama on Linux/macOS or with `strip=True, convert=False` (e.g. redirected to a file or non-tty pipe):
Writing text containing ANSI OSC sequences (such as terminal title changes `\033]0;title\007` or `\033]2;title\007`) invoked `AnsiToWin32.write_and_convert()`, which routed through `convert_osc()`.
In `convert_osc()`, the OSC sequence was stripped, but it unconditionally attempted to execute:
```python
if params[0] in '02':
    winterm.set_title(params[1])
```
Because `winterm` is `None` on non-Windows platforms (and `self.convert` is `False` when stripping without converting), this crashed with:
`AttributeError: 'NoneType' object has no attribute 'set_title'`.

## 3. Reproduction Test Case
Added in `colorama/tests/ansitowin32_test.py`:
```python
    @patch('colorama.ansitowin32.winterm', None)
    def test_convert_osc_without_convert(self):
        s = StringIO()
        stream = AnsiToWin32(s, strip=True, convert=False)
        stream.write('abc\033]2;my title\007def')
        self.assertEqual(s.getvalue(), 'abcdef')
```

Prior to the fix:
`AttributeError: 'NoneType' object has no attribute 'set_title'`

## Solution Approach
1. **Isolate Platform-Specific Win32 Calls:** In `AnsiToWin32.convert_osc()`, `winterm.set_title()` was invoked even when `convert=False` (or `winterm is None` on Linux/macOS) whenever `strip=True` encountered an OSC title sequence, raising `AttributeError: 'NoneType' object has no attribute 'set_title'`.
2. **Guard Win32 Title Updates Behind `convert` & `winterm`:** Added a check ensuring `winterm.set_title()` is only called when `self.convert` is enabled and `winterm` is initialized, while still stripping the OSC sequence cleanly when `strip=True`.
3. **TDD & Regression Safety:** Added `test_convert_osc_without_convert` in `colorama/tests/ansitowin32_test.py` verifying OSC sequences are stripped safely on non-Windows environments.

## 4. Solution & Code Changes
In `colorama/ansitowin32.py`:
- In `convert_osc()`: Guard the `winterm.set_title` call with `if params[0] in '02' and self.convert and winterm:`. When `convert=False` or `winterm is None`, the OSC sequence is cleanly stripped from the output stream without invoking win32 methods.
- In `call_win32()`: Added a guard `elif winterm is None: return` before dispatching to winterm screen/cursor methods to guard against NoneType calls on non-Windows environments.

## 5. Verification
- Reproduction test passed.
- All 39 test cases across `colorama/tests/` passed (`.venv/bin/pytest`).
