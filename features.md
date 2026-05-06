# CMMS (Maintenance Management) — Feature & Functionality Survey

> Candidate #237 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| IBM Maximo Application Suite | Enterprise EAM/CMMS | Commercial SaaS / On-prem (Red Hat OpenShift) | https://www.ibm.com/products/maximo |
| SAP Plant Maintenance (PM) | ERP-integrated CMMS module | Commercial (SAP S/4HANA licensing) | https://www.sap.com/products/erp/s4hana.html |
| Fiix (Rockwell Automation) | Cloud-native CMMS | Commercial SaaS | https://fiixsoftware.com/ |
| Limble CMMS | Mid-market CMMS | Commercial SaaS | https://limble.com/ |
| UpKeep | Mobile-first CMMS | Commercial SaaS | https://upkeep.com/ |
| MaintainX | Digital maintenance platform | Commercial SaaS | https://www.getmaintainx.com/ |
| eMaint (Fluke) | Condition-monitoring CMMS | Commercial SaaS | https://www.emaint.com/ |
| Infor EAM (HxGN EAM) | Enterprise EAM | Commercial SaaS / On-prem | https://www.infor.com/products/eam |
| Atlas CMMS (Grashjs) | Open-source self-hosted CMMS | Open source (MIT) | https://github.com/Grashjs/cmms |
| openMAINT | Open-source EAM/CMMS | Open source (LGPL) | https://www.openmaint.org/ |

---

## Feature Analysis by Solution

### IBM Maximo Application Suite

**Core features**
- Work order creation, assignment, tracking, and closure with full audit trail
- Preventive maintenance scheduling (calendar, meter, and condition-based triggers)
- Asset register with full lifecycle management from procurement to disposal
- MRO inventory and spare parts management with automated reordering
- IoT/sensor data ingestion from PLCs, SCADA systems, and IIoT devices
- AI-powered predictive maintenance via IBM watsonx (anomaly detection, failure forecasting)
- Generative AI natural-language interface for asset queries and report generation
- Compliance and audit trail management (FDA, OSHA, GMP)
- Multi-site and multi-tenant enterprise deployment

**Differentiating features**
- IBM watsonx generative AI integration for natural-language maintenance interactions
- Maximo Monitor: unified OT/IT data plane ingesting SCADA, PLC, and sensor streams
- Asset performance management (APM) module with condition-based maintenance scoring
- Red Hat OpenShift deployment supporting on-prem, hybrid, and cloud configurations
- Industry-specific accelerators (Oil & Gas, Utilities, Transportation, Healthcare)

**UX patterns**
- Traditional enterprise UI; functional but complex; requires significant training
- Role-based dashboards targeting maintenance managers, planners, and technicians
- Mobile app for field technicians (work order execution, asset inspection)
- Progressive disclosure via module licensing — organisations buy only modules they need

**Integration points**
- REST API (IBM Maximo REST API Guide): https://ibm-maximo-dev.github.io/maximo-restapi-documentation/
- Node.js SDK on GitHub: https://github.com/ibm-maximo-dev/maximo-nodejs-rest-client
- Native SAP, Oracle, and ERP connectors
- IoT: MQTT, OPC UA, Maximo Monitor data ingestion
- IBM API Hub: https://developer.ibm.com/apis/catalog/maximo--maximo-manage-rest-api/

**Known gaps**
- Extremely high total cost of ownership; typically $500k–$5M+ implementation
- Long deployment timelines (12–24 months for enterprise implementations)
- Steep learning curve; dedicated Maximo administrators required
- Limited accessibility for small-to-mid market organisations

**Licence / IP notes**
- Proprietary commercial software; IBM licensing
- No open-source components in core product

---

### SAP Plant Maintenance (PM)

**Core features**
- Equipment master and functional location hierarchy management
- Notification creation (malfunction, activity, general maintenance request)
- Work order management: corrective, preventive, refurbishment, and inspection order types
- Maintenance planning including annual maintenance plans and strategy-based scheduling
- Bill of materials (BOM) for equipment spare-parts planning
- Integration with SAP Materials Management (MM) for procurement and inventory
- Integration with SAP HR for technician workforce management
- Audit-trail compliance reports for OSHA, FDA, and GMP

**Differentiating features**
- Deep native integration with SAP ERP modules (MM, PP, HR, FI/CO)
- Maintenance cost centre accounting with direct cost allocation
- Equipment capacity planning aligned to production planning (PP module)

**UX patterns**
- Traditional SAP transaction-based UI (T-codes); steep learning curve
- Fiori-based mobile interface available (newer deployments)
- Heavy reliance on SAP BASIS administrators for configuration

**Integration points**
- SAP APIs (OData/REST) via SAP BTP Integration Suite
- Native integration to SAP S/4HANA modules (MM, PP, FI, HR)
- SAP IoT integration via SAP Edge Services

**Known gaps**
- Requires the full SAP ecosystem — impractical as a standalone CMMS
- Very expensive; tied to SAP licensing and implementation partners
- Rigid data model; customisation requires ABAP development

**Licence / IP notes**
- Proprietary SAP commercial licence
- No open-source components

---

### Fiix (Rockwell Automation)

**Core features**
- Work order management with configurable request forms, task lists, and SOP attachments
- Preventive maintenance scheduling: date/time, meter, event, and condition-based triggers
- Nested PMs and multi-asset work orders
- Asset register with asset hierarchies, QR/barcode scanning
- MRO spare parts inventory with automated reorder
- Fiix Foresight: AI parts demand forecasting analysing historical data
- Fiix Copilot: AI assistant for natural-language queries over assets and CMMS data
- Reporting and KPI dashboards (MTTR, MTBF, PM compliance rate)
- Mobile app (iOS and Android)

**Differentiating features**
- Fiix Foresight parts forecaster: predicts when, what, and how many parts to order
- Fiix Copilot AI: natural-language interface returning risk scores, uptime, and work-order history
- Deep integration with Rockwell Automation factory-floor ecosystem

**UX patterns**
- Modern, clean SaaS UI with strong mobile-first design
- Guided onboarding and asset import wizards
- Progressive disclosure: lite mode for technicians, full mode for planners

**Integration points**
- REST API (Enterprise plan only): https://fiixlabs.github.io/api-documentation/
- Java examples on GitHub: https://github.com/fiixlabs/fiix-cmms-api-java-examples
- Python client: https://github.com/Saint-Solutions/python-fiix-cmms-client
- 1,000+ integrations via no-code connectors (Zapier, Power Automate)
- Rockwell Automation FactoryTalk native connector

**Known gaps**
- REST API requires Enterprise plan; adds cost for integration use cases
- Limited support for very complex asset hierarchies (multi-level parent/child)
- Reporting customisation can be restrictive compared to enterprise tools

**Licence / IP notes**
- Proprietary commercial SaaS; Rockwell Automation subsidiary
- No open-source components

---

### Limble CMMS

**Core features**
- Work order creation, assignment, and tracking with drag-and-drop calendar scheduling
- Preventive maintenance automation with time and meter-based triggers
- Asset tracking with full maintenance history, QR codes, and cost tracking
- Spare parts inventory with min/max thresholds and automated reorder alerts
- Mobile app with offline mode (auto-sync on reconnect)
- AI PM Builder: AI-generated preventive maintenance schedules
- Resource planning: technician workload visibility and assignment optimisation
- Multi-currency reporting for international operations
- 24/7 US-based support

**Differentiating features**
- AI PM Builder (Summer 2025 release): AI generates PM schedules from asset data and failure history
- Resource planning module for workload balancing
- Consistently highest user-adoption ratings in the mid-market segment (fast setup, intuitive UX)

**UX patterns**
- Consumer-grade UX; one of the fastest CMMS implementations in the market
- Drag-and-drop work order scheduling calendar
- Mobile-first technician interface with offline capability

**Integration points**
- REST API v2: https://apidocs.limblecmms.com/
- Postman workspace available
- Regional API endpoints (US, Canada, Australia, Europe, 21CFR)
- ERP connectors (SAP, Oracle, Infor)
- IoT sensor platform connectors
- SSO (SAML)

**Known gaps**
- Lighter analytics than enterprise tools; limited cross-site multi-criteria work order sorting
- MRO inventory filtering across sites less powerful than IBM Maximo or Infor
- Customer support response times reported as inconsistent at scale
- Limited RCM/reliability engineering workflows

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### UpKeep

**Core features**
- Work order management (create, assign, track, close) with photo attachments and voice clips
- Preventive maintenance scheduling
- Asset tracking with lifecycle history and QR/barcode scanning
- Spare parts and inventory management
- Nova AI agent: background monitoring, data quality flagging, automated work-order generation from technician notes, voice-first field access
- Real-time AI-powered insights and KPI dashboards
- Mobile-first design (iOS and Android)

**Differentiating features**
- Nova AI agent: autonomous background monitoring and work-order generation
- Voice-first field interface for hands-free technician use
- Lowest entry-level price point ($20/user/month) among feature-complete CMMS tools

**UX patterns**
- Mobile-first; designed for field technicians above planners
- Clean consumer-grade UI with minimal training required
- Voice clips and photo attachments on work orders

**Integration points**
- REST API (Enterprise only): https://developers.onupkeep.com/
- Webhooks and Zapier for mid-tier plans
- 100+ pre-built connectors (Slack, SAP, etc.)
- Bidirectional SAP sync limited and requires custom API work

**Known gaps**
- REST API and workflow automation gated to Enterprise plan
- Bidirectional SAP ERP sync problematic; limited native ERP integration depth
- Limited support for large multi-site enterprise operations
- Downtime tracking and purchase order management Enterprise-only
- Retroactive correction of closed work orders not supported

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### MaintainX

**Core features**
- Work order management with real-time commenting, checklists, photo/voice attachments, and required fields
- Pre-filled work order templates with technician signature capture
- Checklist and inspection tools with customisable forms and audit trails
- Anomaly detection for work-order error prevention
- Preventive maintenance scheduling
- Spare parts and inventory management
- MaintainX CoPilot: AI assistant for work-order generation, procedure building, and manual extraction (add-on)
- Integration marketplace (Oracle, Infor, Microsoft Dynamics 365, SAP, MachineMetrics, Slack)
- Open REST API and webhooks

**Differentiating features**
- Best-in-class checklist and inspection procedure documentation
- Automatic work-order creation from failed inspection steps
- Native integration marketplace with major ERPs
- Strong mobile UX for deskless/field maintenance workers

**UX patterns**
- Consumer-grade SaaS UI optimised for deskless workers
- Voice clips and photo capture from mobile
- Guided procedure building for managers; simplified execution for technicians

**Integration points**
- REST API v1: https://api.getmaintainx.com/v1/docs
- Bearer token authentication
- Webhooks: workorder.created, workorder.updated, workorder.status_changed, workorder.completed events
- Native integrations: Oracle, Infor, Microsoft Dynamics 365, SAP, Zapier, MachineMetrics, Google Forms, Slack

**Known gaps**
- Analytics capabilities still maturing compared to IBM Maximo or Infor
- Limited reliability-engineering (FMEA, RCM) workflows
- CoPilot AI is an add-on rather than core
- No native IoT sensor ingestion; relies on partner integrations (e.g., MachineMetrics)

**Licence / IP notes**
- Proprietary commercial SaaS
- No open-source components

---

### eMaint (Fluke/Fortive)

**Core features**
- Work order management with PM scheduling
- Asset register and lifecycle tracking
- MRO inventory management
- Condition monitoring module: sensor threshold alerting → automatic work-order creation
- eMaint AI: machine learning vibration-data analysis identifying misalignment, imbalance, looseness, and bearing wear
- SCADA, PLC, RTU, BMS, MES/MOM production data ingestion
- 1,000+ low-code integrations (NetSuite, Salesforce, Power BI, SAP, Oracle)
- Fluke Connect wireless sensor integration (vibration, temperature)

**Differentiating features**
- Tightest native condition-monitoring integration in mid-market (Fluke instrument ecosystem)
- SCADA/OT data ingestion at a price point below IBM Maximo
- eMaint AI for rotating machinery fault classification (four major fault types)
- Accelix: unified CMMS + SCADA + Condition Monitoring data plane

**UX patterns**
- Web-based with standard mid-market CMMS UI patterns
- Condition monitoring dashboard surfacing sensor health alongside work orders

**Integration points**
- REST API v2: https://api.x4.emaint.com/
- JWT authentication (OAuth 2 Bearer Token)
- Fluke Connect and Fluke Reliability sensor platform native integration
- Siemens, Tulip, and Ignition SCADA connectors
- 1,000+ apps via low-code connectors

**Known gaps**
- UI is less modern than Fiix or Limble
- Mid-tier feature depth outside condition monitoring; less suited for complex asset hierarchies
- Pricing opaque (contact for quote)
- Limited AI beyond rotating-machinery fault classification

**Licence / IP notes**
- Proprietary commercial SaaS; Fluke/Fortive subsidiary
- No open-source components

---

### Infor EAM (HxGN EAM)

**Core features**
- Enterprise asset register with hierarchical functional locations
- Work order management: work management module combining workforce and work-order tracking
- Preventive maintenance planning and scheduling (calendar, meter, condition-based)
- Condition-based PM with IoT sensor integration (vibration, temperature, pressure)
- Coleman AI: predictive failure analytics on historical work-order and sensor data
- Reliability and risk management modules (FMEA, asset criticality ranking)
- Inventory, warranty, and strategic planning modules
- Asset failure forecasting with reasons and performance reporting
- Multi-industry compliance (FDA 21 CFR, OSHA PSM, GMP)

**Differentiating features**
- Deep reliability engineering tooling: FMEA, RCM support, criticality analysis
- Coleman AI predictive analytics tightly integrated with work-order data
- Multi-industry compliance breadth (Oil & Gas, Pharmaceutical, Manufacturing, Utilities)

**UX patterns**
- Complex enterprise UI; requires dedicated configuration and training
- Role-based dashboards for maintenance managers, reliability engineers, and planners
- Mobile field execution apps

**Integration points**
- REST APIs (Infor OS integration platform)
- Native Infor CloudSuite ERP integration
- IoT sensor connectors
- SAP and Oracle connectors via middleware

**Known gaps**
- Complex initial configuration; long implementation timelines
- High cost comparable to IBM Maximo for full reliability module suite
- UI less modern than mid-market competitors
- Limited self-service customisation

**Licence / IP notes**
- Proprietary commercial SaaS / on-prem; Hexagon subsidiary
- No open-source components

---

### Atlas CMMS (Grashjs/cmms)

**Core features**
- Work order management with mobile app support
- Preventive maintenance scheduling
- Asset tracking and lifecycle management
- Spare parts inventory control
- Reporting and analytics
- Multi-language support
- API integrations

**Differentiating features**
- Fully self-hosted under MIT licence; no vendor lock-in
- GPL v3 / MIT licence enables unrestricted modification and redistribution
- Free to use with no seat limits for self-hosted deployments

**UX patterns**
- Standard web CMMS UI
- Mobile app for field execution

**Integration points**
- REST API available
- GitHub: https://github.com/Grashjs/cmms

**Known gaps**
- Community-supported only; no professional support SLA
- Limited AI/ML capabilities
- No native IoT or SCADA integration
- Feature depth significantly below commercial enterprise tools

**Licence / IP notes**
- Open source — MIT licence
- No IP restrictions; free to fork, modify, and redistribute

---

### openMAINT

**Core features**
- Asset and facilities management (CAFM/CMMS hybrid)
- Work order management
- Preventive maintenance scheduling
- Asset register with full lifecycle tracking
- Building and spatial management (integration with BIM models)

**Differentiating features**
- BIM/spatial integration for facilities and real-estate use cases
- LGPL licence enables commercial embedding without copyleft obligations

**UX patterns**
- Web-based UI; dated compared to modern SaaS tools
- Configurable via graphical administration tools

**Integration points**
- REST API
- BIM/IFC model import
- GitHub: https://gitlab.com/openmaint/

**Known gaps**
- Limited AI capabilities
- Small community; slower release cadence
- Facilities/real-estate focus; less suited to industrial manufacturing

**Licence / IP notes**
- Open source — LGPL
- Free to embed in commercial products without copyleft obligations on the host application

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Work order creation, assignment, tracking, and closure
- Preventive maintenance scheduling (time-based and meter-based triggers)
- Asset register with maintenance history and QR/barcode scanning
- Spare parts / MRO inventory management with reorder alerts
- Mobile app for field technician execution (iOS and Android)
- KPI dashboards (MTTR, MTBF, PM compliance rate)
- Role-based access control
- Basic ERP integration (SAP, Oracle, Microsoft Dynamics)
- Audit trail for regulatory compliance (OSHA, FDA, GMP)

### Differentiating Features
- AI-powered predictive maintenance (sensor anomaly detection, failure forecasting)
- Generative AI / natural-language interface for work-order generation and data queries
- Condition-based maintenance triggered by live IoT/sensor thresholds
- OT data ingestion (SCADA, PLC, RTU, MES) beyond simple sensor connections
- Reliability engineering modules (FMEA, RCM, criticality analysis)
- AI spare-parts demand forecasting
- Voice-first field interface for hands-free technician use
- BIM/spatial integration for facilities management

### Underserved Areas / Opportunities
- Affordable AI-native predictive maintenance for mid-market and SMB (currently locked to enterprise pricing)
- True bidirectional ERP sync without custom API development
- Natural-language procedure generation from equipment manuals and failure history
- Automated root-cause analysis correlating sensor data, work-order history, and failure modes
- Sustainability tracking linking maintenance activities to energy consumption and emissions reporting
- Multi-criteria work-order prioritisation using AI (combining criticality, downtime risk, parts availability)
- Unified IoT + CMMS open-source platform (the open-source tools lack IoT; commercial IoT is expensive)

### AI-Augmentation Candidates
- Predictive failure detection: rules-based threshold alerting → ML anomaly detection on sensor streams
- Work-order generation: manual failure reporting → AI-triggered from sensor anomalies
- PM schedule optimisation: fixed calendar intervals → AI-optimised condition-based schedules
- Root-cause analysis: manual expert review → AI correlation of failure modes, asset history, operating context
- Spare-parts forecasting: manual reorder points → AI demand forecasting with failure-probability weighting
- Procedure generation: manual SOP authoring → AI generation from equipment manuals and technician notes
- Natural-language work-order querying: report builder → LLM conversational interface over CMMS data

---

## Legal & IP Summary

All major commercial CMMS vendors (IBM Maximo, SAP PM, Fiix, Limble, UpKeep, MaintainX, eMaint, Infor EAM) are proprietary closed-source platforms protected by vendor copyright and software licences. No open-source components from these platforms can be reused. Open-source alternatives (Atlas CMMS under MIT, openMAINT under LGPL) are freely usable and modifiable without licence restrictions. The MIT licence on Atlas CMMS is the most permissive and allows unrestricted commercial use. No patent concerns were identified in the publicly available information reviewed; however, specific AI/ML methods used by IBM watsonx, Fiix Foresight, and eMaint AI may be subject to patent protection. Independent implementation of similar AI techniques using published research methods carries no known IP risk.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Work order management: create, assign, track, close with photo/file attachments and audit trail
- Preventive maintenance scheduling with time-based and meter-based triggers
- Asset register with maintenance history, QR/barcode scanning, and hierarchical locations
- Spare parts inventory with reorder alerts and usage tracking
- Mobile app (offline-capable) for field technician execution
- REST API with webhook support for ERP and IoT integration
- Role-based access control and compliance audit trail

**Should-have (v1.1)**
- AI-powered anomaly detection on connected IoT/sensor data streams → automatic work-order creation
- Natural-language AI assistant for work-order querying and basic report generation
- AI PM schedule optimisation (condition-based interval recommendations from asset history)
- Bidirectional ERP sync (SAP, Oracle, Microsoft Dynamics) without manual data entry
- KPI dashboards with MTTR, MTBF, PM compliance, and downtime cost tracking

**Nice-to-have (backlog)**
- AI-driven root-cause analysis correlating sensor anomalies, failure modes, and work-order history
- AI spare-parts demand forecasting with failure-probability weighting
- Natural-language procedure generation from equipment manuals and historical technician notes
- Reliability engineering modules (FMEA, criticality ranking, RCM analysis support)
- Sustainability reporting linking maintenance activities to energy use and emissions
- BIM/spatial integration for facilities management use cases
