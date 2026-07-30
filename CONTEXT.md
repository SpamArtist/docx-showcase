# DocX Showcase

The public portfolio context that demonstrates DocX to recruiters through curated evidence, a controlled live analysis experience, and a transparent build narrative without exposing the private DocX implementation.

## Language

**DocX Showcase**:
The recruiter-facing single-page portfolio and controlled public analysis experience.
_Avoid_: DocX website, marketing site, report viewer

**Portfolio Report**:
A public-safe projection of a DocX report containing only approved recruiter-facing sections, measurements, and redacted evidence.
_Avoid_: Public DocX report, full report, audit report

**Sample Portfolio Report**:
The immediate, version-labeled Portfolio Report for FX Inline that demonstrates the experience without requiring a visitor to submit an extension.
_Avoid_: Demo data, fake report, default report

**Live Analysis**:
The one-time DocX analysis of the current version of a visitor-submitted Chrome Web Store extension when no completed Portfolio Report already exists for that version.
_Avoid_: Live scan, rerun, URL scan

**Analysis Job**:
The bounded asynchronous work that acquires one extension version, analyzes it without executing extension code, projects a Portfolio Report, and removes the acquired artifacts.
_Avoid_: Scan request, background task, worker run

**Extension Version**:
The immutable analysis identity formed from a canonical Chrome Web Store extension ID and its exact resolved version.
_Avoid_: Extension URL, extension, listing

**Sanitized Report Snapshot**:
The retained Portfolio Report for one Extension Version after public-field allowlisting and redaction.
_Avoid_: Cached raw report, report cache, analysis output

**In-Session Report**:
A Portfolio Report rendered only within the visitor's current single-page interaction, without a permanent report URL or public report directory.
_Avoid_: Private report, shareable report, report page

**Source Map Analysis**:
The public-safe summary of DocX source-map attribution capability that excludes mappings, embedded sources, and complete original paths.
_Avoid_: Source maps, source-map contents, reconstructed source

**Scan Allowance**:
The visitor-level limit applied to accepted uncached Extension Versions; viewing an existing Sanitized Report Snapshot does not consume it.
_Avoid_: API quota, report limit, request throttle

**Public Build Story**:
The sanitized, self-contained account of the Wayfinder decisions, architecture, red-green-refactor delivery, measurements, and selected code excerpts behind the DocX Showcase.
_Avoid_: Development log, private issue mirror, full build history
