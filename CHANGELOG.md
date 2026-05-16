# Changelog

All notable changes to the DAT specification will be documented in this file.

## [0.1.0-draft] - 2026-05-08

### Added

- Initial specification draft (originally published as AIT — Agent Identity Token)
- Three token types: DAT-S (Session), DAT-A (Action), DAT-D (Delegation)
- Chain verification model with human authority at root
- JSON Schemas for all token types
- Cryptographic requirements (RS256, SHA-256)
- Scope model with glob patterns and deny lists
- Security considerations (context substitution, replay, scope escalation)
- Standard action types table

### Changed

- Renamed from AIT (Agent Identity Token) to DAT (Depwire Action Token) to avoid collision with IETF AgentID draft (May 2026)
