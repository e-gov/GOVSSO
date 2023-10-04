---
permalink: Mock
---

<img src='img/eu_regional_development_fund_horizontal.jpg' width="350" height="200" alt="European Union European Regional Development Fund"/>

# Mock for GovSSO service

GovSSO mock is an application that serves [GovSSO protocol](TechnicalSpecification) to clients. Its main use cases are:

* Enable client applications to develop and test integration with GovSSO protocol. Compared to GovSSO service demo
  environment (`govsso-demo.ria.ee`), mock can be used without needing registration with RIA and can also be used
  offline or in closed networks.
* Provide mock authentication data in development and test environments. Compared to GovSSO service demo environment
  (`govsso-demo.ria.ee`), mock can return arbitrary user data to client application and is also simpler to use with
  automated tests.

NB! GovSSO mock currently validates most conditions on input data, therefore not all validations in GovSSO mock are
currently as strict as in GovSSO environment.

GovSSO mock can be found at [https://github.com/e-gov/GOVSSO-Mock](https://github.com/e-gov/GOVSSO-Mock).
