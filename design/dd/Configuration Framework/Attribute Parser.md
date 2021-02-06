Attribute Parser
===

## Introduction
This feature is introduced from [[Inject Attribute to Prototype Service]]

To inject attribute to a Prototype Service, the Service Framework introduce an annotation `InitAttribute`, the annotation support reference value, Configuration Framework will support `config` prefix reference value like `config:config path`, the attribute parser will try find a configuration from specified config path.

## Implementation
![[Inject Attribute to Prototype Service#Attribute Parser APIs]]

```mermaid
classDiagram

class ConfigAttributeParser {
	<<Service>>
}
IAttributeParser <.. ConfigAttributeParser

class ConfigurationSupport {
	<<Service>>
	-_rootConfiguration Configuration
}
IServiceSupport <.. ConfigurationSupport
ConfigurationSupport "1" *-- "1" ConfigAttributeParser
ConfigurationSupport "1" *-- "1" ConfigValueParsers
ConfigurationSupport "1" *-- "1" ConfigSatisfyHook

class ConfigValueParsers {
	<<Service>>
}
ConfigValueParsers "1" *-- "*" IConfigValueParser

class ConfigSatisfyHook {
	<<Service>>
}
ISatisfyHook <.. ConfigSatisfyHook
```