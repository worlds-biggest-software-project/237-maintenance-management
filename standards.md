# Standards & API Reference

> Project: CMMS (Maintenance Management) · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISO Standards

**ISO 55000:2024 — Asset Management: Vocabulary, Overview and Principles**
- URL: https://www.iso.org/standard/83053.html
- Provides the foundational vocabulary, principles, and overview for asset management systems. A CMMS is the primary operational tool for demonstrating conformance; the standard defines what an asset management system must achieve, making it the baseline design reference for any CMMS.

**ISO 55001:2024 — Asset Management: Asset Management System Requirements**
- URL: https://www.iso.org/standard/83054.html
- The certifiable management system standard specifying requirements for establishing, implementing, maintaining, and improving an asset management system. A properly configured CMMS is the data backbone required to achieve ISO 55001 certification. Organisations using a CMMS typically achieve certification readiness in 6–12 months vs. 18–24 months without one.

**ISO 13306:2017 — Maintenance: Maintenance Terminology**
- URL: https://www.iso.org/standard/70306.html
- Defines standardised maintenance terminology including failure, fault, breakdown, corrective maintenance, preventive maintenance, condition-based maintenance, and predictive maintenance. Informs consistent work-order type classification, failure-mode coding, and maintenance-action categorisation within any CMMS data model.

**ISO 14224:2016 — Petroleum, Petrochemical and Natural Gas Industries: Collection and Exchange of Reliability and Maintenance Data for Equipment**
- URL: https://www.iso.org/standard/64076.html
- Defines taxonomy, data collection methodology, and failure-mode classification for reliability and maintenance data. Widely adopted across process industries globally for benchmarking equipment reliability. A CMMS targeting Oil & Gas, Utilities, or Chemical sectors should align its data model and failure-coding schema to ISO 14224.

**ISO 15926 — Integration of Life-Cycle Data for Process Plants Including Oil and Gas Production Facilities**
- URL: https://www.iso.org/standard/56849.html
- A framework standard for the integration, sharing, and handover of lifecycle data for process plant assets using Semantic Web technology. Relevant for CMMS implementations requiring data interchange with capital-project systems, EPC contractors, and owner-operators in the process industries.

**ISO 31000:2018 — Risk Management: Guidelines**
- URL: https://www.iso.org/standard/65694.html
- Provides principles and guidelines for risk management applicable to maintenance risk assessments, criticality analysis, and FMEA processes often embedded in or linked to CMMS reliability modules.

---

### W3C & IETF Standards

**RFC 7230–7235 — Hypertext Transfer Protocol (HTTP/1.1)**
- URL: https://tools.ietf.org/html/rfc7230
- The foundational HTTP specification underpinning all REST API communication between CMMS platforms and external systems. All CMMS REST APIs (Fiix, Limble, UpKeep, MaintainX, eMaint, IBM Maximo) are built on HTTP/1.1 semantics.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://tools.ietf.org/html/rfc6749
- The standard for delegated authorisation used by CMMS API authentication. eMaint uses OAuth 2 Bearer Token; most SaaS CMMS platforms implement OAuth 2.0 or API-key flows derived from it.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://tools.ietf.org/html/rfc7519
- JWT is used by eMaint X4 API v2 for access tokens and is increasingly adopted across CMMS REST APIs as the standard token format for stateless authentication.

**RFC 8288 — Web Linking**
- URL: https://tools.ietf.org/html/rfc8288
- Defines link relations used in REST API hypermedia responses; relevant for CMMS APIs supporting pagination and resource linking (HATEOAS patterns).

**W3C Web of Things (WoT) — Thing Description**
- URL: https://www.w3.org/TR/wot-thing-description/
- W3C standard for describing IoT device capabilities and interfaces in a machine-readable format. Relevant when a CMMS incorporates an IoT integration layer that needs to discover and describe connected sensors and equipment.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (fka Swagger)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for documenting REST APIs. Limble CMMS API v2 (https://apidocs.limblecmms.com/) and MaintainX REST API v1 (https://api.getmaintainx.com/v1/docs) publish OpenAPI-compatible documentation. Any new CMMS REST API should provide an OpenAPI 3.1 specification.

**JSON Schema**
- URL: https://json-schema.org/
- Standard for describing and validating JSON data structures. Used extensively in CMMS API request/response validation, webhook payload schemas, and IoT sensor data models.

**MQTT v5.0 (ISO/IEC 20922)**
- URL: https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- Lightweight publish-subscribe messaging protocol standard for IoT sensor data transmission to CMMS condition-monitoring layers. MQTT is the dominant protocol for IIoT sensor-to-cloud communication and is used in conjunction with CMMS platforms (eMaint, IBM Maximo Monitor, Limble IoT connectors).

**OPC UA (IEC 62541) — OPC Unified Architecture**
- URL: https://opcfoundation.org/developer-tools/documents/view/161
- The primary industrial interoperability standard (IEC 62541) for exchanging data between PLCs, SCADA systems, and higher-level systems such as CMMS and ERP. OPC UA provides standardised data models for 60+ equipment types via Companion Specifications. IBM Maximo, eMaint, and Infor EAM all support OPC UA as an OT data ingestion pathway.

**MQTT Sparkplug B**
- URL: https://www.eclipse.org/tahu/spec/Sparkplug_Topic_Namespace_and_State_ManagementV2.2-with-ESF_Required_Changes_Spec.pdf
- Open standard extending MQTT with consistent topic namespaces and payload encoding for industrial equipment data. Increasingly used alongside OPC UA to deliver structured sensor telemetry to CMMS condition-monitoring modules.

**ISO 15926 / CFIHOS (Capital Facilities Information Handover Specification)**
- URL: https://www.iogp.org/bookstore/product/capital-facilities-information-handover-specification/
- CFIHOS is an IOGP-developed extension of ISO 15926 defining how information is structured and handed over between project phases in capital-facilities projects. Directly relevant for CMMS data models that must accept structured asset handover packages from EPC contractors in Oil & Gas, Petrochemical, and Power generation sectors.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and Bearer Token (RFC 6750)**
- URL: https://tools.ietf.org/html/rfc6750
- Standard for API access delegation and bearer-token authentication. Used by eMaint (OAuth 2 Bearer), Limble (Basic/API Key), and UpKeep (API Key derived from OAuth flows). Any new CMMS API should implement OAuth 2.0 with short-lived JWT access tokens.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0 providing standardised user authentication and SSO. Limble supports SAML-based SSO; enterprise CMMS products broadly support OIDC/SAML for enterprise identity provider integration (Okta, Azure AD, Ping Identity).

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- Widely used SSO standard for enterprise CMMS deployments integrating with corporate identity providers. Limble, IBM Maximo, Infor EAM, and Fiix all support SAML 2.0 SSO.

**OSHA PSM — 29 CFR 1910.119 (Process Safety Management of Highly Hazardous Chemicals)**
- URL: https://www.osha.gov/laws-regs/regulations/standardnumber/1910/1910.119
- US federal regulation requiring documented mechanical integrity programs for process equipment containing highly hazardous chemicals. A CMMS serving Oil & Gas, Chemical, or Petrochemical customers must support PSM-compliant work-order workflows, inspection documentation, and audit trails.

**FDA 21 CFR Part 11 — Electronic Records and Signatures**
- URL: https://www.accessdata.fda.gov/scripts/cdrh/cfdocs/cfcfr/CFRSearch.cfm?CFRPart=11
- FDA regulation governing electronic records and signatures in pharmaceutical, biotech, and medical-device manufacturing. A CMMS serving these sectors must provide Part 11-compliant audit trails, electronic signatures on work orders, and access controls. Limble provides a dedicated 21CFR API endpoint region.

**NIST SP 800-53 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- NIST framework for security controls. Relevant for CMMS deployments in US federal, defence, and critical-infrastructure contexts (utilities, transportation, water treatment).

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- De-facto checklist for API security design. Any CMMS REST API should be audited against the OWASP API Security Top 10 (Broken Object Level Authorisation, Broken Authentication, Excessive Data Exposure, etc.).

---

### MCP Server Specifications

Model Context Protocol (MCP) is directly relevant to an AI-native CMMS. An MCP server exposing CMMS data (assets, work orders, maintenance history, sensor readings) would allow AI agents and LLM-based interfaces to query, create, and update maintenance records using natural language. Key MCP resources:

**Model Context Protocol — Official Documentation**
- URL: https://modelcontextprotocol.io/docs
- Defines the protocol for exposing tools and resources to AI models. A CMMS MCP server could expose: `get_asset`, `list_work_orders`, `create_work_order`, `get_maintenance_history`, `get_sensor_readings`, `update_asset_status` as MCP tools.

**MCP Server SDK (TypeScript / Python)**
- URL: https://github.com/modelcontextprotocol/sdk
- Official SDK for building MCP servers in TypeScript and Python. The most practical starting point for building an MCP-native CMMS interface.

---

## Similar Products — Developer Documentation & APIs

### IBM Maximo Application Suite

- **Description:** Enterprise EAM/CMMS platform with IoT, AI (IBM watsonx), and multi-site asset management. Gold standard for critical-infrastructure and complex industrial environments.
- **API Documentation:** https://ibm-maximo-dev.github.io/maximo-restapi-documentation/
- **SDKs/Libraries:** Node.js SDK — https://github.com/ibm-maximo-dev/maximo-nodejs-rest-client
- **Developer Guide:** https://developer.ibm.com/apis/catalog/maximo--maximo-manage-rest-api/
- **Standards:** REST/JSON; OSLC (Open Services for Lifecycle Collaboration); OData
- **Authentication:** OAuth 2.0 / API Key; SAML 2.0 SSO

---

### Fiix CMMS (Rockwell Automation)

- **Description:** Cloud-native mid-market CMMS with AI Copilot and Foresight parts forecasting. Strong mobile UX and Rockwell Automation ecosystem integration.
- **API Documentation:** https://fiixlabs.github.io/api-documentation/
- **SDKs/Libraries:**
  - Java examples: https://github.com/fiixlabs/fiix-cmms-api-java-examples
  - Python client: https://github.com/Saint-Solutions/python-fiix-cmms-client
- **Developer Guide:** https://fiixlabs.github.io/api-documentation/guide.html
- **Standards:** REST/JSON; API available to Enterprise plan subscribers
- **Authentication:** API Key (Enterprise plan required)

---

### Limble CMMS

- **Description:** Mid-market CMMS with highest adoption ratings, AI PM Builder, and comprehensive REST API with regional endpoints including a dedicated 21 CFR-compliant region.
- **API Documentation:** https://apidocs.limblecmms.com/
- **SDKs/Libraries:** Postman workspace — https://www.postman.com/limbleapiqa/limble-solutions-llc-s-public-workspace/documentation/zskh2o7/limble-api-v2
- **Developer Guide:** https://help.limblecmms.com/en/collections/3322827-api
- **Standards:** REST/JSON; OpenAPI-compatible documentation
- **Authentication:** Basic Auth / API Key; regional endpoints: US, Canada, Australia, Europe, 21CFR

---

### UpKeep

- **Description:** Mobile-first CMMS with Nova AI agent for autonomous work-order generation. Lowest entry price; REST API gated to Enterprise plan.
- **API Documentation:** https://developers.onupkeep.com/
- **SDKs/Libraries:** None officially published
- **Developer Guide:** https://upkeep.com/integrations/rest-api/
- **Standards:** REST/JSON
- **Authentication:** API Key (Enterprise plan required); Webhooks and Zapier on mid-tier plans

---

### MaintainX

- **Description:** Digital maintenance platform optimised for deskless workers; strong checklist/inspection tooling with open REST API and webhook event system.
- **API Documentation:** https://api.getmaintainx.com/v1/docs
- **SDKs/Libraries:** Make (Integromat) connector — https://apps.make.com/maintainx
- **Developer Guide:** https://help.getmaintainx.com/zapier-integration/retrieve-maintainx-data-with-zapier
- **Standards:** REST/JSON; OpenAPI-compatible
- **Authentication:** Bearer Token (API Key); Webhook events: workorder.created, workorder.updated, workorder.status_changed, workorder.completed

---

### eMaint CMMS (Fluke/Fortive)

- **Description:** Condition-monitoring CMMS with native Fluke instrument integration and SCADA/OT data ingestion. Automated work-order creation from sensor threshold breaches.
- **API Documentation:** https://api.x4.emaint.com/
- **SDKs/Libraries:** Python client (community): https://github.com/rlouch2/eMaintAPIClient; Siemens Senseye connector: https://developer.siemens.com/senseye/workorders/pull/emaint.html
- **Developer Guide:** https://www.emaint.com/emaint-cmms-api-unites-your-individual-business-processes/
- **Standards:** REST/JSON; JWT (OAuth 2 Bearer Token); OData-style query filters
- **Authentication:** JWT access tokens; OAuth 2.0 Bearer Token; TLS on port 443 (host: x46.emaint.com)

---

### Atlas CMMS (Grashjs/cmms)

- **Description:** Free, self-hosted open-source CMMS under MIT licence. Full source code available; REST API included. Suitable as a starting point or reference implementation.
- **API Documentation:** https://github.com/Grashjs/cmms (see API docs in repo)
- **SDKs/Libraries:** None; community contributions welcome
- **Developer Guide:** GitHub README
- **Standards:** REST/JSON
- **Authentication:** JWT; standard RBAC

---

### openMAINT

- **Description:** Open-source CMMS/CAFM with BIM integration under LGPL licence. Suitable for facilities management and real-estate asset management use cases.
- **API Documentation:** https://gitlab.com/openmaint/ (repository)
- **SDKs/Libraries:** None
- **Developer Guide:** https://www.openmaint.org/en/documentation
- **Standards:** REST/JSON; IFC/BIM model import
- **Authentication:** Session-based / API Key

---

## Notes

- **OT/IT convergence gap:** Most commercial CMMS REST APIs are designed for IT-layer integration (ERP, HR, analytics). OT-layer integration (SCADA, PLC, RTU) requires specialist middleware (Accelix/eMaint, IBM Maximo Monitor, or third-party IIoT platforms). An AI-native CMMS should plan for a native OT integration layer from the outset.
- **Open standards for IoT:** The OPC UA + MQTT Sparkplug B combination has emerged as the leading open-standard pairing for IIoT sensor data delivery to CMMS platforms in 2026. New implementations should prioritise these over proprietary sensor connectors.
- **AI-specific standards:** No mature ISO or W3C standard exists yet for AI-generated maintenance recommendations or predictive maintenance model governance. NIST AI RMF (https://www.nist.gov/system/files/documents/2023/01/26/AI%20RMF%201.0.pdf) is the leading framework for managing AI risk in industrial applications and should be referenced for AI feature governance.
- **MCP emerging:** Model Context Protocol is rapidly maturing as the standard interface for exposing domain data to AI agents. Early adoption of MCP for CMMS data exposure would position an AI-native CMMS significantly ahead of incumbents whose AI interfaces are proprietary (Fiix Copilot, MaintainX CoPilot, IBM watsonx).
