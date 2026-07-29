# Kentico digital experience

Extend the digital marketing features of Xperience by Kentico with custom components, written by your AI coding assistant.

Marketers configure these features using the component types available to them. When they need behavior Xperience doesn't provide out of the box, such as posting to a chat channel, calling an internal service, or branching a process on data Xperience doesn't track, a developer adds a custom component. This plugin hands that work to your coding assistant. You explain what the component does and which settings marketers control, and the agent writes the implementation together with the registration that makes it available in the admin UI.

## Choose a skill

| Skill | Use it to |
|---|---|
| `automation-action` | Implement and register a custom automation action, with optional marketer-configurable properties |
| `automation-condition` | Implement and register a custom automation condition that branches a process, with optional marketer-configurable properties |

The two skills cover the two kinds of custom step. An action *does* something for the contact that reaches it. A condition *decides* which of two paths that contact takes, and changes nothing.

For the other kinds of automation customization, see the [customization overview](https://docs.kentico.com/x/automation_custom_xp).

> [!TIP]
> New to agent skills? Agents activate skills as necessary based on the assigned task. Alternatively, you can use slash commands and other methods depending on your assistant. For what that means in practice, read [Invoke a skill](../../docs/Usage-Guide.md#invoke-a-skill).

## Requirements

- An Xperience by Kentico project with [Automation](https://docs.kentico.com/x/automation_xp) in use
- An AI coding assistant with this plugin installed
- The [Documentation MCP server](https://docs.kentico.com/x/mcp_server_xp), configured as described in [MCP setup](./MCP-setup.md)
- A description of what the step does, and of any settings marketers need to change

Additionally, `automation-condition` requires:

- A description of the branch logic — what makes the condition true, what data it reads, and which path each outcome leads to

## Install

Follow the marketplace instructions in the [usage guide](../../docs/Usage-Guide.md#install-the-selected-plugin), using the plugin name `kentico-digital-experience`.

## Build your first action

This sequence produces one working action, configurable by marketers. See the other sections for more use cases and prompt examples.

1. Open the solution containing your Xperience web project in your AI coding assistant.

2. Describe the step and its settings in the prompt. The more specific you are, the fewer questions the agent needs to ask. For examples, see [Write effective prompts](#write-effective-prompts).

   ```text
   /automation-action

   Create an action that sends a Slack message to a configured webhook
   when a contact reaches this step. Marketers need to edit the webhook
   URL and the message template.
   ```

3. Answer the design questions. The agent reads the current Xperience documentation and studies how your project is organized, then confirms the design before writing anything. Expect it to ask about whatever you left open, such as how the step behaves when the external call fails.

4. Review the generated code. The agent reports which files it created. Read [Review the output](#review-the-output).

5. Build the project and restart the application, then open the **Automation** application and add your step to a process. Confirm the step appears under its display name and that its properties render in the configuration dialog.

At this point the step is available to every marketer working in the Automation Builder, and it runs for each contact that reaches it.

## Build your first condition

This sequence produces one working condition that marketers can drop into a process to split it in two.

1. Open the solution containing your Xperience web project in your AI coding assistant.

2. Describe what makes the condition true and what data it reads. State which outcome belongs on which branch if it isn't obvious.

   ```text
   /automation-condition

   Create a condition that is true when the contact has an active
   subscription in our billing service. Marketers need to pick which
   subscription tier counts as active.
   ```

3. Answer the design questions. The agent reads the current Xperience documentation and studies how your project is organized, then confirms the design before writing anything. Expect it to ask what should happen when the data it needs is missing.

4. Review the generated code. The agent reports which files it created. Read [Review the output](#review-the-output).

5. Build the project and restart the application, then open the **Automation** application and add your condition to a process. Confirm it appears under its display name, that its properties render in the configuration dialog, and that both the true and the false branch lead where you expect.

## Common tasks

### Add an automation action with no settings

The agent skips the properties class entirely.

```text
/automation-action

Add an action that writes an information-level log entry with the
contact's email address. No marketer-facing settings.
```

### Drive the configuration from a requirements file

Point the agent at examples, specification documents, or an existing implementation. The agent reads the source instead of your summary of it, which produces a closer match.

```text
/automation-action

Implement the action described in ./requirements/crm-sync.md and follow
the conventions of the components already in this project.
```

### Add a property to an existing action

Describe the action and the setting. The agent finds the existing classes and extends them rather than starting over.

```text
Add a retry-count setting to the CrmSyncAction, editable by marketers,
with a default of 3 and a maximum of 10.
```

### Add an automation condition with no settings

The agent skips the properties class entirely.

```text
/automation-condition

Add a condition that is true when the contact's email address uses one
of our internal domains. No marketer-facing settings.
```

### Branch on data produced by an earlier step

Name the action that stores the data. The agent wires the condition to read the same process data type instead of fetching the value again.

```text
/automation-condition

Add a condition that is true when the lead score stored by the
LeadScoringAction is above a threshold the marketer sets.
```

### Add a property to an existing condition

Describe the condition and the setting. The agent finds the existing classes and extends them rather than starting over.

```text
Add a country setting to the HasActiveSubscriptionCondition, editable by
marketers as a dropdown, so the check can be limited to one market.
```

## Write effective prompts

Both of these prompts produce a working output. However, providing the agent with more context significantly reduces guesswork and increases output quality and standards adherence. Compare:

| Prompt | What happens |
|---|---|
| `Create an action that sends an email` | The agent works out the recipient, the source of the content, and which parts marketers control. Expect several rounds of questions. |
| `Create an action that sends the contact a transactional email chosen by the marketer from a dropdown of published email templates` | The agent proposes a design right away and asks only about what you left open. |
| `Create a condition that checks the contact's subscription` | The agent works out where the subscription data lives, what counts as a match, and which outcome belongs on the true branch. Expect several rounds of questions. |
| `Create a condition that is true when the contact has an active subscription of a tier the marketer picks from a dropdown, read from our billing API` | The agent proposes a design right away and asks only about what you left open. |

> [!TIP]
> The same applies to every KentiCopilot skill. See [Write specific prompts](../../docs/Usage-Guide.md#write-specific-prompts) for the general guidance, including the habits that slow a session down.

## Review the output

Treat generated code the way you'd treat a pull request from someone new to the project. The following things are worth reviewing:

**Form annotation namespace** -- The [form component](https://docs.kentico.com/x/8ASiCQ) attributes that build the configuration dialog need to come from the `Kentico.Xperience.Admin.*.FormAnnotations` namespaces. An obsolete Form Builder namespace, `Kentico.Forms.Web.Mvc`, contains attributes with the same names. Check the `using` directives on the properties class.

**Execution time limit** -- Actions are cancelled after two minutes, so a step calling a slow external service can be cut off mid-run. If the generated code talks to anything outside the application, read [Best practices](https://docs.kentico.com/x/automation_custom_steps_xp) for the timeout behavior and what to do instead.

**Side effects in a condition** -- A condition evaluates and returns a result, nothing more. The same contact can be evaluated more than once, so anything the generated `Evaluate` method writes, sends, or stores happens repeatedly. Move that work into an action.

**The false branch as the failure path** -- A condition that times out or throws resolves to `false`, which makes the false branch the path contacts take when something goes wrong, not only when the answer is genuinely no. Check that the branch is a safe place to land.

Other things to keep an eye on during review:

- Use `ILogger<T>` for [logging](https://docs.kentico.com/documentation/developers-and-admins/development/logging).
- Check the failure path, and what happens when the same contact enters the step twice.
- Consider [data protection](https://docs.kentico.com/x/zIB1CQ) issues for anything the step sends outside the application.
- Open the configuration dialog in the **Automation** application and verify the output.

## Customize

Record project-specific conventions in your project's agent instruction files. The agent otherwise infers coding conventions from surrounding code each time. Instructions kept in the project apply to every task and survive plugin updates.

> [!TIP]
> Durable project context, exploring before generating, and verifying against the running site all improve task outcomes. See [Work effectively with KentiCopilot](https://docs.kentico.com/x/work_effectively_kenticopilot_guides) for details.
