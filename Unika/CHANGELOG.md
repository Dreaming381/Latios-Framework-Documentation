# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.14.1] – 2025-11-2

Officially supports Entities [1.3.14]

### Fixed

-   Fixed C\#10 file scope namespaces not being handled correctly in source
    generators

## [0.14.0] – 2025-10-18

Officially supports Entities [1.3.14]

This release only contains internal compatibility changes

## [0.12.5] – 2025-4-12

Officially supports Entities [1.3.9]

### Fixed

-   Fixed `ScriptRef` resolving on buffers that had scripts removed such that
    the number of scripts in the buffer exactly equals what used to be index of
    the script referenced by `ScriptRef`

## [0.12.4] – 2025-4-5

Officially supports Entities [1.3.9]

### Added

-   Added `DynamicBuffer<UnikaScripts>` extension method `CopyScriptFrom()`

## [0.12.0] – 2025-2-23

Officially supports Entities [1.3.9]

### This is the first release of *Unika*.
