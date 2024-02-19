---
permalink: Example
---

<img src='img/eu_regional_development_fund_horizontal.jpg' width="350" height="200" alt="European Union European Regional Development Fund"/>

# Example client for integrating with GovSSO service

One example of client implementation is provided for integrating with GovSSO in Java. Its source can be found at [https://github.com/e-gov/TARA-GovSSO-ExampleClient](https://github.com/e-gov/TARA-GovSSO-ExampleClient). It demonstrates all flows that a GovSSO client must implement (authentication, session update, logout, and back-channel logout). OpenID Connect support that is needed for GovSSO integration is based on the Spring Security framework's OAuth 2.0 module. Example client source code is provided for study purposes and it cannot be used out of the box in production.

For demonstration and testing purposes, example client deployments are publicly accessible as described on the [Demo](Demo) page.
