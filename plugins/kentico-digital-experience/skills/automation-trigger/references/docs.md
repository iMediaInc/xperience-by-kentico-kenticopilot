# Documentation links

Fetch any link via the Kentico Docs MCP. Read only what the task needs.

## Automation customization

- **Custom automation triggers**: <https://docs.kentico.com/documentation/developers-and-admins/digital-marketing-setup/automation-customization/automation-custom-triggers>
  - When to read: always, first. The authoritative reference — base classes, `Evaluate`, trigger data, trigger properties, registration, dispatching from code, time-based triggers, and best practices (statelessness, timeouts, data guidelines, versioning).
- **Custom automation steps**: <https://docs.kentico.com/documentation/developers-and-admins/digital-marketing-setup/automation-customization/automation-custom-steps>
  - When to read: when a step inside the process has to consume the trigger data, or when the task turns out to need a custom action rather than a trigger.
- **Automation customization overview**: <https://docs.kentico.com/documentation/developers-and-admins/digital-marketing-setup/automation-customization>
  - When to read: to see what kinds of custom automation components exist and where triggers fit.
- **Automation overview**: <https://docs.kentico.com/documentation/business-users/digital-marketing/automation>
  - When to read: to understand the marketer's mental model — how a trigger starts a process, recurrence settings, contact mapping — before designing a trigger.

## Firing the trigger

- **Contacts API examples**: <https://docs.kentico.com/api/digital-marketing/contacts>
  - When to read: to obtain the `ContactInfo` to dispatch against — the current request's contact or one retrieved by query.
- **Handle global events**: <https://docs.kentico.com/documentation/developers-and-admins/customization/handle-global-events>
  - When to read: when the trigger fires from an object or content event rather than from application code you control.
- **Scheduled tasks**: <https://docs.kentico.com/documentation/developers-and-admins/customization/scheduled-tasks>
  - When to read: for time-based triggers — a task loads the matching contacts and fires the trigger for each. Docs recommend running such a task at most once per day.

## Admin UI form components (for the properties class)

- **Form components reference**: <https://docs.kentico.com/documentation/developers-and-admins/customization/extend-the-administration-interface/ui-form-components/reference-admin-ui-form-components>
  - When to read: when picking editing components for trigger properties.
- **Validation rules**: <https://docs.kentico.com/documentation/developers-and-admins/customization/extend-the-administration-interface/ui-form-components/ui-form-component-validation-rules.html>
  - When to read: when constraining property values declaratively or defining a custom rule.
- **Visibility conditions**: <https://docs.kentico.com/documentation/developers-and-admins/customization/extend-the-administration-interface/ui-form-components/ui-form-component-visibility-conditions.html>
  - When to read: when a property should show or hide based on another property's value.
- **Configure editing component state**: <https://docs.kentico.com/documentation/developers-and-admins/customization/extend-the-administration-interface/ui-form-components/editing-components/configure-editing-component-state>
  - When to read: when cross-field dependencies need a component configurator — e.g. a dropdown whose options depend on another property.
