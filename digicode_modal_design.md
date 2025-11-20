# Digicode Modal Design Specification

## 1. Overview
This document outlines the design for a PIN pad (Digicode) modal integration into the `VersatileThermostatUi` card. This feature intercepts the unlock action when a lock code is configured, requiring the user to enter a 4-digit PIN to unlock the thermostat.

## 2. State Management
New reactive state properties required in `VersatileThermostatUi` class:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `showDigicodeModal` | `boolean` | `false` | Controls visibility of the modal overlay. |
| `enteredCode` | `string` | `""` | Stores the currently entered digits. Max length 4. |
| `codeError` | `boolean` | `false` | Indicates if the last attempt failed (for visual feedback). |

## 3. Integration Logic

### Modification to `_handleLockToggle`
The existing `_handleLockToggle` method will be updated to check for the `lock_code` attribute before calling the unlock service.

**Logic Flow:**
1.  Check if `allow_lock_toggle` config is enabled (existing).
2.  Check if `isLocked` is true.
    *   **IF** `isLocked` is true **AND** `stateObj.attributes.lock_code` is true:
        *   Set `showDigicodeModal = true`.
        *   Clear `enteredCode`.
        *   Return (do not call service yet).
    *   **ELSE** (standard unlock or locking):
        *   Proceed with existing service call (`versatile_thermostat.unlock` or `versatile_thermostat.lock`).

## 4. UI Component Structure (LitElement)

### Render Method Update
The `render()` method will be updated to include the modal template conditionally based on `showDigicodeModal`. It should be placed at the end of the `ha-card` content, before the closing tag, to ensure it overlays correctly.

### HTML Template
The modal should leverage `ha-dialog` (available in Home Assistant environment) for consistency, handling the backdrop and centering automatically.

```html
<ha-dialog
  .open=${this.showDigicodeModal}
  @closed=${this._handleModalClose}
  heading="Enter Code"
  hideActions
  class="digicode-dialog"
>
  <div class="dialog-content">
    <!-- Code Display -->
    <div class="code-display ${this.codeError ? 'error' : ''}">
      <!-- Render 4 placeholders/dots with filled state logic -->
      <span class="digit ${this.enteredCode.length > 0 ? 'filled' : ''}"></span>
      <span class="digit ${this.enteredCode.length > 1 ? 'filled' : ''}"></span>
      <span class="digit ${this.enteredCode.length > 2 ? 'filled' : ''}"></span>
      <span class="digit ${this.enteredCode.length > 3 ? 'filled' : ''}"></span>
    </div>

    <!-- Keypad Grid -->
    <div class="keypad">
      <div class="keypad-row">
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('1')}>1</mwc-button>
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('2')}>2</mwc-button>
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('3')}>3</mwc-button>
      </div>
      <div class="keypad-row">
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('4')}>4</mwc-button>
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('5')}>5</mwc-button>
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('6')}>6</mwc-button>
      </div>
      <div class="keypad-row">
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('7')}>7</mwc-button>
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('8')}>8</mwc-button>
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('9')}>9</mwc-button>
      </div>
      <div class="keypad-row">
        <div class="keypad-btn spacer"></div>
        <mwc-button outlined class="keypad-btn" @click=${() => this._handleKeypadPress('0')}>0</mwc-button>
        <mwc-button outlined class="keypad-btn action-btn clear" @click=${this._handleKeypadClear}>
            CLEAR
        </mwc-button>
      </div>
    </div>
  </div>
</ha-dialog>
```

## 5. Event Handlers

| Handler | Logic |
|---------|-------|
| `_handleKeypadPress(key)` | 1. If `enteredCode` length < 4, append `key` to `enteredCode`.<br>2. Reset `codeError` to false.<br>3. **If `enteredCode` length is 4**, automatically trigger `_handleValidate()`. |
| `_handleKeypadClear()` | 1. Clear `enteredCode` (set to empty string).<br>2. Reset `codeError`. |
| `_handleModalClose()` | 1. Set `showDigicodeModal = false`.<br>2. Clear `enteredCode`.<br>3. Reset `codeError`. |
| `_handleValidate()` | 1. Check if `enteredCode` length is 4.<br>2. **IF** valid length:<br>   a. Call `versatile_thermostat.unlock` service with `{ code: this.enteredCode }`.<br>   b. Close modal (`_handleModalClose`).<br>3. **ELSE**: Set `codeError = true` (trigger shake animation). |

## 6. CSS Class Considerations

The CSS should ensure the modal content is centered and responsive. Since `ha-dialog` handles the overlay, we focus on the content layout.

```css
/* Dialog Customization */
ha-dialog {
    --mdc-dialog-min-width: 400px; /* Increased width for better button spacing */
    --mdc-dialog-max-width: 450px;
    --mdc-theme-primary: var(--primary-color); /* Theme compatibility */
    --mdc-theme-surface: var(--card-background-color);
}

/* Content Container */
.dialog-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px 0;
}

/* Display Area */
.code-display {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 200px;
  padding: 20px 0;
  border-bottom: 1px solid var(--secondary-text-color);
  margin-bottom: 40px;
}

.digit {
  width: 12px;
  height: 12px;
  margin: 0 5px;
  border-radius: 50%;
  background-color: var(--secondary-text-color);
  opacity: 0.3;
  transition: all 0.2s ease;
}

.digit.filled {
  opacity: 1;
  background-color: var(--primary-text-color);
  transform: scale(1.2);
}

.code-display.error {
  animation: shake 0.4s cubic-bezier(.36,.07,.19,.97) both;
  color: var(--error-color);
}

/* Keypad */
.keypad {
  display: flex;
  flex-direction: column;
  gap: 16px;
  width: 100%;
  max-width: 320px; /* Restrain width for keypad */
}

.keypad-row {
  display: flex;
  justify-content: space-between;
  gap: 16px;
}

.keypad-btn {
  flex: 1;
  --mdc-shape-small: 8px; /* Rectangular with rounded corners like screenshot */
  --mdc-typography-button-font-size: 20px;
  --mdc-typography-button-font-weight: 500;
  height: 56px; /* Larger touch target */
  margin: 0;
  
  /* Theme colors */
  --mdc-theme-primary: var(--primary-color);
  --mdc-button-outline-color: var(--primary-color);
}

.keypad-btn.spacer {
    visibility: hidden;
    pointer-events: none;
}

.keypad-btn.action-btn {
   font-size: 14px; /* Smaller font for CLEAR */
   --mdc-typography-button-font-weight: 700;
}

/* Active state styling handled by mwc-button component */

/* Animations */
@keyframes shake {
  10%, 90% { transform: translate3d(-1px, 0, 0); }
  20%, 80% { transform: translate3d(2px, 0, 0); }
  30%, 50%, 70% { transform: translate3d(-4px, 0, 0); }
  40%, 60% { transform: translate3d(4px, 0, 0); }
}