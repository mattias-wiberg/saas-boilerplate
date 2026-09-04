# SaaS Starter Domain

This context defines the vocabulary for the reusable SaaS application starter and its implementation-ready blueprint.

## Language

**User**:
An authenticated person who uses an application built from the starter.
_Avoid_: Account, customer, workspace member

**Tenant**:
The user-owned boundary for application data in the baseline starter. The baseline has one tenant per user.
_Avoid_: Organization

**Provider**:
An external service used by an application or its operating environment, such as Polar, Resend, or Dokploy.
_Avoid_: Vendor, integration
