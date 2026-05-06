# CMMS (Maintenance Management)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source computerised maintenance management system covering preventive maintenance scheduling, work order management, and asset tracking.

CMMS is a maintenance management platform for manufacturing plants, facilities teams, utilities, and healthcare operators. It targets the gap between extremely expensive enterprise EAM suites (IBM Maximo, SAP PM, Infor EAM) and lighter mid-market SaaS tools, providing AI-native predictive maintenance and natural-language workflows in a self-hostable package.

---

## Why CMMS (Maintenance Management)?

- Enterprise EAM platforms such as IBM Maximo and Infor EAM regularly cost $500k–$5M+ to implement and take 12–24 months to deploy, putting AI-driven predictive maintenance out of reach for mid-market operators.
- SAP Plant Maintenance is impractical as a standalone CMMS — it requires the full SAP ecosystem and ABAP customisation for any data-model change.
- Mid-market SaaS tools (Fiix, UpKeep, MaintainX) gate REST APIs, workflow automation, and bidirectional ERP sync to Enterprise plans, limiting integration depth.
- Existing open-source options (Atlas CMMS under MIT, openMAINT under LGPL) lack AI capabilities and native IoT/SCADA ingestion.
- AI-driven predictive maintenance is becoming a baseline expectation in 2026 rather than a premium feature, but is currently locked to enterprise pricing tiers.

---

## Key Features

### Work Order & Preventive Maintenance

- Work order creation, assignment, tracking, and closure with photo, voice, and file attachments
- Full audit trail for OSHA, FDA 21 CFR Part 11, and GMP compliance
- Preventive maintenance scheduling with time-based, meter-based, event, and condition-based triggers
- Drag-and-drop calendar scheduling and nested PMs across multi-asset work orders
- Checklists, inspection forms, and SOP attachments with technician signature capture

### Asset & Inventory Management

- Asset register with hierarchical functional locations and full lifecycle tracking
- QR and barcode scanning for in-field asset identification
- MRO spare parts inventory with min/max thresholds and automated reorder alerts
- Maintenance history, cost tracking, and warranty management per asset
- Multi-site and multi-currency support for international operations

### AI-Native Capabilities

- Anomaly detection on connected IoT/sensor data streams with automatic work-order creation
- Natural-language assistant for work-order querying and report generation
- AI-optimised PM schedule recommendations derived from asset history and failure data
- AI-driven root-cause analysis correlating sensor anomalies, failure modes, and work-order history
- Spare-parts demand forecasting weighted by failure probability
- Natural-language procedure generation from equipment manuals and historian technician notes

### Mobile & Field Execution

- Offline-capable mobile app for iOS and Android
- Voice clips and photo capture from the field
- Role-based interfaces for technicians, planners, and reliability engineers
- Real-time KPI dashboards: MTTR, MTBF, PM compliance, downtime cost

### Integration & Compliance

- REST API with webhook support for ERP and IoT integration
- Bidirectional ERP sync targets for SAP, Oracle, and Microsoft Dynamics 365
- IoT/OT ingestion via MQTT, OPC UA, and SCADA/PLC connectors
- Role-based access control and compliance audit trail
- Multi-industry compliance support (OSHA PSM, FDA 21 CFR Part 11, GMP, ISO 55000/55001, ISO 14224)

---

## AI-Native Advantage

Incumbents treat AI as a premium add-on (IBM watsonx for Maximo, Fiix Foresight, MaintainX CoPilot, Coleman AI for Infor) priced for the enterprise. This project makes AI core: predictive failure detection on sensor streams, AI-triggered work-order generation from anomalies, AI-assisted root-cause correlation across failure history, and natural-language procedure generation from equipment manuals. Combined with conversational querying over CMMS data, this brings capabilities currently confined to seven-figure deployments to mid-market and SMB operators.

---

## Tech Stack & Deployment

The project targets self-hosted, cloud, and hybrid deployment, drawing on the deployment patterns of IBM Maximo (on-prem/hybrid via OpenShift) and the self-hosting model of Atlas CMMS. Integration is REST-API-first with webhook events for work order lifecycle changes (created, updated, status_changed, completed), aligned to industry standards including ISO 55000/55001 (asset management), ISO 13306 (maintenance terminology), ISO 14224 (reliability and maintenance data), and SAE JA1011/JA1012 for RCM analysis outputs. IoT/OT ingestion uses MQTT and OPC UA for SCADA, PLC, and RTU connectivity.

---

## Market Context

The global CMMS market is projected to reach approximately $2.4 billion in 2026, expanding to $5.9 billion by 2036 at a 9.3% CAGR (Future Market Insights, 2026). Enterprise CMMS contracts (IBM Maximo, SAP, Infor) run from hundreds of thousands to millions of dollars; mid-market SaaS pricing ranges from $16/user/month (MaintainX) to $75/user/month, with REST APIs typically gated to Enterprise tiers. Primary buyers are maintenance managers, reliability engineers, facilities directors, plant managers, and IT/OT integration leads across manufacturing, healthcare, hospitality, utilities, and commercial real estate.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
