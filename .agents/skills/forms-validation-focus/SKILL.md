---
name: forms-validation-focus
description: Text input, focus/first responder, validation UX, and error presentation rules for UIKit/AppKit.
metadata:
  short-description: Forms, validation, and focus rules.
---

# Forms, Validation, and Focus

Use this skill when building forms, text input flows, or validation UX on UIKit or AppKit.

## Rules
- Use system controls only: UITextField/UITextView and NSTextField/NSSecureTextField.
- Input state and validation logic live in side controllers.
- View controllers wire UI, forward input, and display validation errors.
- Validate on submit, then validate on subsequent edits for failed fields.
- Use inline error labels under fields (system colors). Hide when valid.
- Preserve accessibility: error labels must be accessible and associated with fields.
- Never block the main thread. UI updates must be on @MainActor.
- Use OSLog/Logger for validation diagnostics if needed; avoid PII.

## UIKit (Text Fields + Focus Chain)
- Use UITextFieldDelegate and textFieldShouldReturn for focus flow.
- Return key types: .next for intermediate fields, .done for final submit.
- After a submit attempt, failed fields revalidate on edit.

UIKit example:

```swift
@MainActor
final class ProfileViewController: UIViewController, UITextFieldDelegate {

    private let controller: ProfileController
    private let nameField = UITextField()
    private let emailField = UITextField()
    private let nameErrorLabel = UILabel()
    private let emailErrorLabel = UILabel()

    init(controller: ProfileController) {
        self.controller = controller
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        nameField.returnKeyType = .next
        emailField.returnKeyType = .done
        nameField.delegate = self
        emailField.delegate = self

        nameErrorLabel.textColor = .systemRed
        emailErrorLabel.textColor = .systemRed
        nameErrorLabel.isHidden = true
        emailErrorLabel.isHidden = true
    }

    func textFieldShouldReturn(_ textField: UITextField) -> Bool {
        if textField === nameField {
            emailField.becomeFirstResponder()
            return false
        }
        view.endEditing(true)
        submit()
        return false
    }

    func textFieldDidChangeSelection(_ textField: UITextField) {
        controller.update(
            name: nameField.text ?? "",
            email: emailField.text ?? ""
        )
        updateErrors()
    }

    private func submit() {
        controller.submit()
        updateErrors()
    }

    private func updateErrors() {
        nameErrorLabel.text = controller.validationError(for: .name)
        emailErrorLabel.text = controller.validationError(for: .email)
        nameErrorLabel.isHidden = nameErrorLabel.text == nil
        emailErrorLabel.isHidden = emailErrorLabel.text == nil
    }
}
```

## AppKit (Text Fields + First Responder)
- Use NSTextFieldDelegate for updates and Return handling.
- Use window?.makeFirstResponder(nextField) for focus chain.
- Validate on submit, then revalidate on edits for failed fields.

AppKit example:

```swift
@MainActor
final class ProfileViewController: NSViewController, NSTextFieldDelegate {

    private let controller: ProfileController
    private let nameField = NSTextField()
    private let emailField = NSTextField()
    private let nameErrorLabel = NSTextField(labelWithString: "")
    private let emailErrorLabel = NSTextField(labelWithString: "")

    init(controller: ProfileController) {
        self.controller = controller
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        nameField.delegate = self
        emailField.delegate = self
        nameErrorLabel.textColor = .systemRed
        emailErrorLabel.textColor = .systemRed
        nameErrorLabel.isHidden = true
        emailErrorLabel.isHidden = true
    }

    func controlTextDidChange(_ obj: Notification) {
        controller.update(
            name: nameField.stringValue,
            email: emailField.stringValue
        )
        updateErrors()
    }

    func controlTextDidEndEditing(_ obj: Notification) {
        guard let field = obj.object as? NSTextField else { return }
        if field === nameField {
            view.window?.makeFirstResponder(emailField)
        } else {
            submit()
        }
    }

    private func submit() {
        controller.submit()
        updateErrors()
    }

    private func updateErrors() {
        nameErrorLabel.stringValue = controller.validationError(for: .name) ?? ""
        emailErrorLabel.stringValue = controller.validationError(for: .email) ?? ""
        nameErrorLabel.isHidden = nameErrorLabel.stringValue.isEmpty
        emailErrorLabel.isHidden = emailErrorLabel.stringValue.isEmpty
    }
}
```
