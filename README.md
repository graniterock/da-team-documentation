# Data & Analytics Team Documentation

This repository contains the comprehensive documentation for GraniteRock's Data & Analytics team, migrated from Confluence to enable version control and collaborative editing.

## Repository Structure

### 📊 Data Products
Business-focused data organization and documentation:
- **`data-products/apex/`** - Apex system data products (GL and Tickets)
- **`data-products/data-by-domain/`** - Data organized by business domain
- **`data-products/data-by-system/`** - Data organized by source system
- **`data-products/data-by-line-of-business/`** - Data organized by business line
- **`data-products/dbt-docs/`** - dbt model documentation
- **`data-products/da-core-production-data-model/`** - Core production data models

### 🏗️ Data Architecture
Technical architecture documentation and system designs:
- **`data-architecture/data-analytics-home/`** - Main team landing page and navigation
- **`data-architecture/oltp-postgres/`** - OLTP Postgres architecture documentation
  - `oltp-postgres-main/` - Main OLTP documentation
  - `apex-oltp/` - Apex system OLTP processes
  - `jde-oltp/` - JDE system OLTP processes  
  - `data-pipelines/` - Data pipeline documentation
  - `initial-setup/` - Setup and configuration guides

### 🔗 Data Integrations
External system connections and data flows:
- **`data-integrations/`** - Integration documentation including DataServ, JDE, and other external systems

### 📋 Templates
Standardized templates for documentation and processes:
- **`templates/`** - Project planning, decision documentation, and meeting notes templates

## Quick Navigation

- **🏠 [Data & Analytics Home](data-architecture/data-analytics-home/README.md)** - Main team documentation hub
- **🎯 [Data by System](data-products/data-by-system/README.md)** - System-organized data overview
- **🏢 [Data by Line of Business](data-products/data-by-line-of-business/README.md)** - Business line data organization
- **🌐 [Data by Domain](data-products/data-by-domain/README.md)** - Domain-specific data documentation
- **🔧 [OLTP - Postgres](data-architecture/oltp-postgres/oltp-postgres-main/README.md)** - Core data architecture
- **🔌 [Data Integrations](data-integrations/dataserv-integration.md)** - External system integrations

## Migration Notes

This content was migrated from Confluence's "Data & Analytics" workspace using the Atlassian MCP server. The repository structure mirrors the original Confluence organization to maintain familiarity while providing the benefits of version control.

## Contributing

Please follow standard Git workflows when contributing to this documentation:

1. Create feature branches for significant changes
2. Use pull requests for review
3. Follow the established directory structure
4. Use the provided templates for consistency

## Original Source

**Confluence Workspace:** https://graniterock.atlassian.net/wiki/spaces/DA/  
**Migration Date:** August 2025  
**Migration Tool:** Atlassian Official Remote MCP Server

---

*This repository serves as the authoritative source for GraniteRock's Data & Analytics team documentation, replacing the original Confluence workspace.*