# Proposal: Leveraging Action Manifests for Better Agent Security

## **Context**
Agents can make mistakes (e.g. flawed reasoning during planning) or can be manipulated (e.g. via prompt injection attacks). This can lead to unintended or harmful outcomes, such as:

- _Rogue actions_: Executing actions that harm users, e.g. opening the front door.
- _Data exfiltration_: Leaking private user information to unauthorized parties.

To mitigate these risks, agents must be able to characterize actions, so that the security implications of invoking an action are readily understandable, both by the agent itself and by any external observer. We propose capturing these action attributes in what we call _action manifests_: structured descriptions capturing such attributes.

Action manifests allow agents to constrain what actions can be invoked in a given context and how actions can be chained together. Here are some examples of security policies we have successfully deployed in production:

- **Coarse-grained access control**: Tool _T_ should only be callable from callers in trusted clients allowlist _A_
- **Data exfiltration prevention:** If _email_tool_ was executed previously, require user confirmation before using _browsing_tool_.
- **Sandboxing:** Application feature _F_ takes prompts from low-level privileges, only allow low-privileged tools <_A_, _B_, _C_> in that context.

These policies may be enforced _deterministically_ as classical security features implemented outside the model, to provide high-assurance guarantees against rogue actions and data exfiltration. They complement any model-based defenses that agents deploy. Alternatively, giving models visibility into action manifests can lead to higher quality, more judicious decision making when making tool calls.

## **Proposal**
The `ToolAnnotations`[https://github.com/modelcontextprotocol/modelcontextprotocol/blob/3ba3181c7779da74b24f0c083eb7055b6fc9d928/schema/2025-03-26/schema.ts#L730] interface is a container for action properties, similar to the mentioned action manifest. We propose to expand this interface with, security-related hints, that can be made available to MCP clients when dynamically loading tools from MCP servers. We have found that the following properties are useful to build security properties in real-world systems:

- **State-changing vs read-only:** Whether calling the action leads to updates/additions/removals in the external environment or in a lasting change to the internal state of stateful actions. 
- **Reversibility:** If an action is state changing, whether the side-effects of calling are not reversible.
- **Authenticated vs incognito:** whether the action requires end user credentials to be propagated to the backend. MCP clients can read this property and make unauthenticated calls to incognito actions.
- **Device capabilities:** Whether the action can command IoT or physical devices.
- **Async/batch:** Whether the action acts on multiple objects or is called asynchronously.
- **1P vs 3P**: Whether the action depends on services that are 3P to itself. Note that this is already captured by the `openWorldHint` property.

The correct granularity/breadth of security properties for actions is a matter of debate. Note that there is a trade off between granularity and maintenability.

## **Concrete Changes**
```typescript
export interface ToolAnnotations {
  title?: string;
  readOnlyHint?: boolean;
  destructiveHint?: boolean;
  idempotentHint?: boolean;
  openWorldHint?: boolean;
  stateChangingHint?: boolean;
  // only meaningful when `stateChanging` is true
  reversibleHint?: boolean;
  incognitoHint?: boolean;
  // perhaps an enum of device capabilities, or just a boolean
  deviceCapabilitiesHint: DeviceCapabilities;
  asyncHint?: boolean;
  batchHint?: boolean;
}
```

Alternatively, create a new `SecurityAnnotations` interface within `ToolAnnotations` where these new properties can be added.

## **Benefits**

- **Action Semantics:** Security properties allow agents to make more sophisticated security decisions and are essential for implementing security policies based on action capabilities. These policies can govern tools and tool chains at runtime, helping to confine the agent's action space. Examples of policies enabled include coarse-grained access control, data exfiltration prevention, and sandboxing.
- **Domain-Specific Capability Confinement:** Allows agent developers to restrict the permissible action space of an agent to be aligned with its intended purpose and operational domain. For example, agents may filter out any state changing actions when loading actions from a 3P MCP server.
- **Observability:** Benefits better observability mechanisms that enable offline investigations for agent behavior, facilitating incident response, detection and response, abuse prevention, and security analysis.
- **High-Assurance User Confirmation for High-Risk Actions:** Allows developers to implement human-in-the-loop user confirmations with a high degree of assurance, _without relying on model behaviors_.

## **Limitations**

- **Manifest drift:** We call manifest drift to situations where manifest attributes go out-of-sync with action capabilities. Manifest attributes are set by action owners at registration-time, but they must be updated in sync with code changes submitted to the action's backend logic. What's more, changes to transitive dependencies of actions could make manifests attributes become outdated without the action owner noticing it. Actions whose manifests have drifted may introduce vulnerabilities or be overly restrictive.
- **Ambiguity:** Ambiguous manifest attributes might lead to vulnerabilities. For example, browsing an arbitrary URL may or may not be a state changing action.
- **Trust:** Like all data coming from untrusted 3P MCP servers, action properties may be used by malicious actors or supply chain attacks to bypass security features or cause harmful agent outcomes. 
