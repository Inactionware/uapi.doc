Log Framework Enhancement
===

# Terminology

# Introduction

Currently there is no Logging framework, the Logging framework just provide a `ILogger` interface to make caller to output log message, the implementation using `slf4j` to output log message to configured logger appender.

For an application, the Logging framework needs provide more functionality but not only used to output log message, for instance for an application needs to output different log content to different file, redirect specific log content to specific file to make easy to check application running status or debug application for specific issue, etc.

# Proposal

