# PREVENT-4

## Description
Configure Enhanced HTTP

## Summary
[Enhanced HTTP](https://learn.microsoft.com/en-us/mem/configmgr/core/plan-design/hierarchy/enhanced-http) (eHTTP) is a simplified method of secure communication without the overhead of a standard PKI deployment. In an eHTTP setup, the site issues self-signed certificates to the various site servers, such as management points and distribution points. Then, these site systems issue unique site tokens to clients. The client then uses the site token for communication with site servers. Microsoft provides a diagram of this process (Figure 1).

![Figure 1](./prevent-4_ehttp-diagram.png)

_Figure 1 - Enhanced HTTP Diagram_

**NOTE:** The preferred/recommended method for secure communication is the use of PKI certificates ([PREVENT-8](../PREVENT-8/prevent-8_description.md)). eHTTP is a compromise between PKI and standard HTTP use and is a more secure option than the latter.

## Linked Defensive IDs
- [PREVENT-8: Require PKI certificates for client authentication](../PREVENT-8/prevent-8_description.md)

## Associated Offensive IDs
- [CRED-2: Request machine policy and deobfuscate secrets](../../../attack-techniques/CRED/CRED-2/cred-2_description.md)

## References
- Microsoft, Enhanced HTTP, https://learn.microsoft.com/en-us/mem/configmgr/core/plan-design/hierarchy/enhanced-http