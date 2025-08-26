## AI Agent Security Action Manifests

Local copy of an early proposal that was submitted to the MCP spec to add extensible security metadata to MCP tools. Basic support for tool hints is now adopted in MCP. The spec now also defines an additional metadata field in the base tool
response type, which allows arbitrary metadata to be attached to tool responses on the server side. Spec-compliant SDKs guarantee that tool metadata (both tool hints & arbitrary metadata) are propagated to clients over the wire. 
Spec-compliant client SDKs should pick these up, but some work is usually required in agent frameworks to make all tool metadata available to the agent & its planning capabilities.
