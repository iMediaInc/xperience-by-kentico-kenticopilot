---
name: "automation-condition"
description: "Reference for implementing custom automation process conditions in Xperience by Kentico. Use whenever the user wants to branch an automation process on custom logic — a class that evaluates every contact reaching the step and sends it down the true or the false path, optionally configurable by marketers through properties."
compatibility: "Requires Kentico Docs MCP"
---

This skill points you to Kentico's automation-customization documentation. Use it to implement a custom automation process condition.

## Pieces of a custom condition

- **Condition class** – inherits `AutomationCondition` (no configuration) or `AutomationCondition<TProperties>` (marketer-configurable); implements the logic in `Evaluate`, which returns `true` for the true branch and `false` for the false branch.
- **Properties class** – implements `IAutomationConditionProperties`; public properties annotated with admin UI form components define the configuration dialog.
- **Registration** – the `RegisterAutomationCondition` assembly attribute (identifier, display name, icon, description) makes the condition appear in the Automation Builder.
- **Runtime context** – `AutomationProcessContext` gives access to the processed contact, the process, and trigger data.
- **Process data** – `IAutomationProcessData` implementations carry typed data from earlier steps; a condition reads that data, it never writes it.

## How to use

- Read `references/docs.md` and fetch the docs pages listed there.

## Gotcha

- Keep `Evaluate` read-only and idempotent — a condition can be re-evaluated for the same contact, so side effects belong in an action.
- Both an unhandled exception and the two-minute timeout resolve the condition to `false`, indistinguishable from a genuine no — handle the failures you can inside `Evaluate`, log them, and return a deliberate result.
- Form-component and validation attributes come from the `Kentico.Xperience.Admin.*.FormAnnotations` namespaces — never from `Kentico.Forms.Web.Mvc`, an obsolete Form Builder namespace with matching class names.
- Use `ILogger<T>` for logging. `EventLogService` is obsolete — don't use it.
