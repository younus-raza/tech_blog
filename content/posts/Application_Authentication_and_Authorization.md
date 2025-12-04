+++
date = '2025-12-02T22:02:31-08:00'
draft = false
title = 'A Practical Guide to Application Authentication and Authorization'
+++

## Introduction 
Authentication (AuthN) answers “Who are you?”
Authorization (AuthZ) answers “What are you allowed to do?”

In today's complex distributed systems, it is crucial to ensure sensitive data is safeguarded, unauthorized access is prevented, and each request is examined with the correct identity and context. Trust cannot be automatically placed in the network or the requester in contemporary systems. Instead, each request must be validated throughout a sophisticated distributed environment, adhering to the foundational concept of the Zero Trust approach.

Moreover, organizations are obligated to adhere to regulatory standards like GDPR and HIPAA, necessitating stringent measures surrounding data security, privacy, and access management. A modern AuthN/AuthZ system should offer Attribute-Based Access Control (ABAC) for detailed, context-sensitive decision-making, ensure non-repudiation by linking every action to a verified identity that cannot be refuted later, and support various identity types including human users, AI agents, and backend systems.

Furthermore, secure propagation of identity across all downstream services is essential to enable each service to grant access based on the attributes of the identity and the resource being accessed. The system should also support Federated Identity and Single Sign-On using contemporary standards like OpenID Connect (OIDC). Additionally, it is imperative for the system to be fully auditable, with each system involved in a request generating standardized access logs that can be consolidated to provide a comprehensive account of who did what, when and under what context.

This article outlines a high-level design for constructing such an AuthN/AuthZ system that adheres to the principles of Zero Trust, caters to modern distributed applications, and offers robust security assurances while being practical to implement.

# Authentication 
The authentication system distinguishes between Human Users, AI Agents, and Backend Services. These identities are provided by external services like Google, Okta, or any system compatible with OpenID Connect. A federation layer, such as Amazon Federate, enables Human Users to select from various identity providers when logging in.

**Human Users** (and AI agents designed to behave like humans) undergo authentication using the standard OpenID Connect Authorization Code Flow: 
1. Users visit our application.
2. They are directed to the chosen Identity Provider.
3. Upon successful authentication, the Identity Provider redirects them back to our application with an ID Token, Access Token, and Refresh Token. 
The ID Token is utilized by the client (browser or mobile app) to display basic profile details. The Access Token and Refresh Token are securely stored as httpOnly, same-site, same-domain cookies to prevent unauthorized access or exfiltration via XSS attacks. Since they are stored in cookies, both tokens are automatically included in every backend request. However, only the Access Token is actively used in normal operations. The Refresh Token is disregarded by the backend until the Access Token expires or becomes invalid. In such cases, the backend starts a secure token refresh process using the Refresh Token to acquire new tokens and continue the request seamlessly. This method safeguards refresh tokens while ensuring uninterrupted sessions.

On the contrary, **Backend Services** do not utilize OpenID Connect for authentication. Instead, they register with our application and obtain a Client ID and Client Secret. Each request from a Backend Service is signed with the Client Secret, following a protocol akin to Amazon Signature Version 4 (AWSv4). The request includes a signature calculated over the payload, headers, and timestamp. 
Upon receiving the request, the backend verifies the signature using the registered Client Secret to confirm that:
- The request originates from a trusted service
- The request has not been tampered with during transit
- Replay attacks are thwarted through timestamp or nonce mechanisms
This setup enables Backend Services to authenticate without user involvement while ensuring robust cryptographic security measures, complementing the token-based authentication for Human Users and AI agents.

![Authentication Flow: Human, AI, and Backend Services](/images/application_security/authentication_flow.svg)

## Transitive Authentication
Our application often needs to communicate with other services once a user or backend service has gone through the authentication process, all while maintaining the identity and context of the original requestor. It is not secure to simply pass the client's Access Token along, as these tokens are meant for communication between the client and application only and should not be shared further downstream. 
To address this concern, we implement a Transitive Authentication Identity Provider (TA-IDP) service. The process functions as follows: When a request is received, the application exchanges the incoming Access Token for a temporary Transitive Authentication Token (TAToken) that has a short lifespan, such as 5 minutes. This TAToken, a JWT containing crucial details like the user's or system's identity and access scopes and use it to authorize the request, is then sent downstream in the request headers. This approach allows each service to verify actions based on the authenticated identity. This approach ensures: 
- Security: The original Access Token is never exposed beyond the application. 
- Continuity of trust: Downstream services can reliably verify the identity of the requester. 
- Non-repudiation: Every action can be traced back to the authenticated identity, establishing a secure audit trail. 
  
By combining TA-IDP with short-lived tokens, the system can securely propagate identity information across services, preventing the misuse of long-lasting credentials and maintaining a strong trust framework for decentralized applications.

![Transitive Authentication Flow](/images/application_security/ta_flow.svg)

## Authorization
In our system, resources, which include objects or data requiring protection, are managed by Attribute-Based Access Control (ABAC) rules. These rules determine access based on various attributes and context, allowing or denying access to resources for specific identities.

Access decisions are made by combining identity attributes (like user ID, roles, and scopes), resource attributes (such as type, owner, other metadata), and contextual information (like time, location, or device). Each resource can have multiple policies that are evaluated collectively to determine access authorization.

To implement ABAC, the ABAC engine is deployed locally within each service to minimize network calls and reduce latency. Policies are centrally defined and managed using a control plane tool, stored in a database, and cached locally within each service. This caching ensures that policies are regularly refreshed for up-to-date access control without affecting performance.

In the access flow, a client sends a request with a TAToken in the header to a service. The service then forwards the TAToken and resource attributes to its local ABAC engine for evaluation. After considering relevant policies and attributes, the engine provides an access decision (allow/deny) which the service enforces before processing the request.

This setup guarantees fine-grained, context-aware access control through multiple resource-specific policies, low-latency enforcement using local ABAC engines, centralized policy management with local caching for consistency, and alignment with Transitive Authentication to ensure verifiable and enforceable identity propagation via TATokens.

![Authorization Flow](/images/application_security/authorization_flow.svg)
![ABAC Policy Management Control Plane Flow](/images/application_security/abac_policy_management.svg)

# Audit Log
Each system that conducts permission checks (such as reviewing TAToken, enforcing ABAC regulations, or implementing access limitations) publishes a security event log. These logs are transferred to a central auditing system or logging tool, collectively establishing a detailed record of security events to aid in:

**Analysis:** Recognizing resource usage patterns, understanding access methodologies, and evaluating policy efficiency.
**Problem Solving:** Pinpointing causes of authorization refusals, token glitches, or service configuration mistakes.
**Compliance & Review:** Offering evidence of who accessed specific resources, at what time, and for what reason.
Detection of Anomalies: Identifying unusual access patterns, potential security breaches, or policy violations.

### Audit Event Flow
When a system receives a request along with a TAToken and a resource plea, the local ABAC mechanism inspects the regulations to determine if the request should be granted or denied. Subsequently, the system compiles a security incident record containing:

- User identity and related specifics
- Details about the resource and its associations
- Versions of the reviewed policies
- Outcome of the permission verdict and the reasoning behind it
- Correlation Trace ID
- Information regarding the system and its operation
  
Then, this report is transmitted to the central audit system.

### Advantages
**Immutable Audit Log:** Each system contributes its portion, linked through Trace IDs.
**Anomaly identification:** The centralized system can run machine learning algorithms or queries to pinpoint unusual trends.

![Security Events Flow](/images/application_security/security_audit.svg)

### What information is collected in Security Event Log?
The Security Access Log Field Descriptions are as follows:

- **Timestamp**: An ISO8601 UTC timestamp indicating when the request was received, essential for maintaining an audit trail and ensuring chronological order.
- **Trace_id**: A unique identifier for the request, facilitating correlation of logs across services in a distributed system.
- **Identity.id**: A distinctive identifier for the user, AI agent, or backend service initiating the request.
- **Identity.type**: The classification of the identity, such as human_user, ai_agent, or backend_service.
- **Identity.metadata**: A collection of contextual attributes related to the identity, like roles, scopes, location, device, and TAToken hash, utilized by ABAC rules.
- **Request.method**: The action performed, for instance, the HTTP method (GET, POST) or RPC operation.
- **Request.endpoint**: The path to the resource or API endpoint that was accessed.
- **Request.source_ip**: The client's IP address that triggered the request, aiding in geographical context and security monitoring.
- **Resource.id**: A unique identifier for the resource that was accessed.
- **Resource.type**: The category or type of the resource, such as document or database_entry.
- **Resource.metadata**: Contextual attributes concerning the resource, like department, classification, and project, evaluated by ABAC policies.
- **Authorization.policies_evaluated**: A list of policies examined for the request, including policy_id and policy_version, crucial for reproducibility.
- **Authorization.decision**: The result of the policy evaluation, either allow or deny.
- **Authorization.reason**: The rationale behind the decision, such as which policy or attribute triggered the allow or deny outcome.
- **Service.name**: The name of the service handling the request, particularly beneficial in auditing multi-service environments.
- **Service.instance_id**: A specific identifier for the service instance processing the request.
- **Service.execution_duration_ms**: The time taken (in milliseconds) to handle the request, aiding in detecting performance irregularities.
  
## Conclusion

![Security Structure for Distributed Applications](/images/application_security/Application_Security_System.svg)
Diagram: Security Structure for Distributed Applications

The visual representation showcases the comprehensive security design of the distributed application. Users, AI entities, and backend functions undergo authentication through OpenID Connect or HMAC-verified requests. The TA-IDP generates temporary TATokens to securely convey identity to subsequent services. Individual services apply ABAC regulations autonomously by utilizing cached directives obtained from the main control hub. Security incidents are recorded by each service and relayed to a centralized auditing mechanism for tracking, adherence to standards, and identification of irregularities. Different colored pathways differentiate between regular request processing, auditing activities, and policy administration to enhance visual comprehension.

This article presents a holistic strategy for managing authentication, authorization, and auditing in contemporary distributed systems by merging Zero Trust principles with practical application tactics. The system's fundamental elements include:

- Verification of identities (AuthN): Differentiating human users, AI entities, and backend services by utilizing OpenID Connect for user identities and HMAC-signed requests for service-to-service communications. The secure administration of Access Tokens and Refresh Tokens is crucial to thwart unauthorized entry and session disruptions.
- Transitive Authentication (TA-IDP): Issuance of short-lived TATokens for downstream service requests to ensure secure identity propagation, uphold non-repudiation, and limit the exposure of enduring credentials.
- Authorization (AuthZ) via Attribute-Based Access Control (ABAC): Local ABAC mechanisms enforce meticulous, context-aware access management by assessing numerous resource-specific policies against identity, resource characteristics, and situational metadata. These policies are centrally controlled and locally cached to decrease response time.
- Recording of Audit Logs: Each system generates security event records that document user identity, resource accessibility, policy assessment, and system context. These logs are aggregated centrally to create immutable audit trails, aid in anomaly detection, assist in troubleshooting, and ensure compliance with regulations.

The accompanying system deployment illustration consolidates all these elements visually, emphasizing:

- User interactions with the application and backend services.
- Transitive authentication pathways via the TA-IDP.
- Implementation of service-level ABAC with local policy caching.
- Segregation of audit logging and control plane management from the primary request flow for clarity.
- Color-coded distinctions for request processing (blue), auditing (green), and policy administration (orange).