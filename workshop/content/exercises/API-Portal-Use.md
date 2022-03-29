API portal for VMware Tanzu enables API consumers to find APIs they can use in their own applications. Consumers can view detailed API documentation and try out an API to see if it will meet their needs. API portal assembles its dashboard and detailed API documentation views by ingesting OpenAPI documentation from the source URLs. An API portal operator can add any number of OpenAPI source URLs to be displayed in a single instance.

API portal can be deployed in a distributed model designed to support scale and self-service:
- Individual teams can deploy their own API portal as they evolve their APIs and associated documentation
- Operators can configure any number of API portal instances in various environments (e.g., dev, test, prod) and provide access to specified groups—such as lines of businesses—based on how groups are organized
- Operators can easily set up API portal with Single Sign-On through their OpenID Connect provider of choice

Alongside **Spring Cloud Gateway for Kubernetes** and Spring Cloud Gateway for VMware Tanzu, API portal provides baseline management capabilities for an internal API sharing economy across an organization's teams and lines of business.