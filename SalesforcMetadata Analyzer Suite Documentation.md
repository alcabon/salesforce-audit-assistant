## Salesforce Metadata Analyzer Suite Documentation

This documentation outlines the key features and capabilities of the various specialized analyzers within the Salesforce Metadata Analyzer Suite. These tools are designed to provide comprehensive insights into code health, metadata usage, performance, and security across a Salesforce organization, leveraging the **Tooling API** and **Metadata API**.

***

## 1. ConsistentDependencyAnalyzer (Apex Code)

The `ConsistentDependencyAnalyzer` provides a **comprehensive, integrated analysis of Apex code**, combining dependency mapping, code coverage, and test quality assessment into a single, cohesive report.

### Key Features

* **Smart Categorization:** Automatically classifies Apex classes as **Production** or **Test** based on naming conventions and selective body analysis (using `@isTest`, `testMethod`, etc.) to optimize memory usage and processing time.
* **Dependency Mapping:** Uses `EnhancedUsageAnalyzer` to identify **used** vs. **unused** Apex classes by analyzing `MetadataComponentDependency` records.
    * **Unused Production Classes** are flagged with a **severity** based on their **Lines of Code** (LoC): `HIGH` (LoC > 500), `MEDIUM` (LoC > 200), or `LOW`.
* **Code Coverage Integration:** Incorporates results from `ApexCodeCoverageAnalyzer` for **Production** classes:
    * Calculates **average coverage percentage**.
    * Categorizes coverage into **Excellent** ($\ge 90\%$), **Good** ($75\%-89\%$), **Fair** ($50\%-74\%$), and **Poor** ($< 50\%$) distribution.
* **Test Quality Assessment:** Analyzes **Test** classes for quality metrics, going beyond simple coverage:
    * Checks for the presence and count of **assertions**.
    * Detects usage of **strong assertions** (`System.assertEquals`, `Assert.areEqual`) versus **weak assertions** (`System.assert(true)`).
    * Identifies best practice usage like `@TestSetup`, data factories, and `Test.startTest()/Test.stopTest()`.
    * Assigns a **Quality Score** (0-100) and **Grade** (A-F).
* **Integrated Recommendations:** Provides actionable advice based on combined findings, such as prioritizing **low-quality tests** for **low-coverage production classes**.

### Output Metrics

| Metric Category | Key Metrics/Findings |
| :--- | :--- |
| **Production Code** | Total Classes, Used Classes, Unused Classes, **Average Code Coverage**, Coverage Distribution. |
| **Code Health Risk** | Critical Unused Fields (for fields), High LoC Unused Classes, Poor Coverage Classes ($\text{Coverage} < 50\%$). |
| **Testing Code** | Total Test Classes, **Average Quality Score**, **Grade Distribution** (A/B/C/D/F), Critical Test Issues (e.g., no assertions). |

***

## 2. EnhancedFieldsAnalyzerWithData

The `EnhancedFieldsAnalyzerWithData` extends metadata analysis by integrating **actual data verification** to determine true field usage and safe removal candidates.

### Key Features

* **Comprehensive Metadata Retrieval:** Fetches detailed metadata for custom and standard fields using the `FieldDefinition` Tooling API object, including security classification, indexing, and object relationships.
* **Data Usage Verification:** Integrates with an assumed `SmartFieldDataAnalyzer` to check for **actual data presence** in records for each field.
    * The data check includes metrics like `nonNullCount`, `totalRecords`, and a **Confidence** level (`HIGH`, `MEDIUM`, `LOW`) based on the sampling `strategy`.
* **Critical Findings:** Highlights fields that pose immediate risk or require urgent cleanup:
    * **`hasDataButUnused`:** Fields that **contain actual data** but are marked as metadata **unused** (CRITICAL - potential data loss risk if removed).
    * **`requiredFieldsEmpty`:** Fields that are **required** but currently **contain no data** (CRITICAL - data integrity issue).
* **Cleanup Candidates:** Identifies **`trulyUnusedFields`** (no metadata references **AND** no actual data) as the safest candidates for removal.
* **Performance Analysis:** Identifies potential performance issues:
    * **`unnecessaryIndexes`**: Indexed fields with **no actual data**.
    * **`recommendedForIndexing`**: Filterable text fields without an index **that contain actual data**.

### Output Metrics

| Metric Category | Key Metrics/Findings |
| :--- | :--- |
| **Data Analysis** | Fields Analyzed, **Fields With Data**, Fields Without Data, Analysis Time, Confidence Distribution (`HIGH`/`MEDIUM`/`LOW`). |
| **Usage Findings** | `trulyUnusedFields`, `hasDataButUnused` (CRITICAL), `requiredFieldsEmpty` (CRITICAL), `calculatedFields`. |
| **Security/Gov** | `unclassifiedFields`, `highSecurityFields`, `fieldHistoryTrackedFields`. |
| **Summary** | Critical Issues Count, Performance Issues Count, **Usage Percentage** (based on data presence). |

***

## 3. EnhancedSecurityAnalyzer

The `EnhancedSecurityAnalyzer` performs a multi-layered security audit, building upon the standard Salesforce Health Check model by adding deeper **user and profile-centric analysis** with quantified **risk scores**.

### Key Features

* **Multi-Phase Analysis:** Runs checks across multiple security domains: Basic Settings, User Permissions, Profile Security, OAuth/Connected Apps, and User Security Patterns.
* **User Permission Analysis:** Specifically queries for high-risk permissions and quantifies the **user count** affected:
    * Users with **"Modify All Data"** (CRITICAL/HIGH risk).
    * Users with **"View All Data"** (MEDIUM risk).
    * **Inactive privileged users** (CRITICAL - dormant high-privilege accounts).
* **Profile Security Deep Dive:** Analyzes specific dangerous permissions *per Profile* (e.g., `PermissionsManageUsers`, `PermissionsCustomizeApplication`), calculating risk based on the permission type and the number of **active users** assigned to that profile.
* **OAuth/API Security:** Checks for:
    * **Unsecured Connected Apps** with multiple tokens.
    * **Old OAuth tokens** ($\text{Older than 180 days}$).
    * Components using **legacy API versions** ($\text{API Version} < 50.0$).
* **Risk-Based Scoring:** Calculates an **`overallScore`** based on compliance and penalties from a weighted **`overallRiskScore`** and **`userImpactPenalty`** (derived from the total number of affected users).

### Output Metrics

| Metric Category | Key Metrics/Findings |
| :--- | :--- |
| **Overall** | **Overall Score** ($\text{0-100}$), Total Checks, Critical Count, Warning Count. |
| **User Impact** | **Total Users Affected**, **Overall Risk Score** (quantified penalty for issues). |
| **User/Profile** | Users with Modify/View All Data, **Profile Security Items** (status: `CRITICAL`/`WARNING` per profile for dangerous permissions). |
| **Settings** | **Session Settings** (e.g., `Lock sessions to domain`), **Certificate Expiration**, **File Upload Security**. |

***

## 4. ApexCodeCoverageAnalyzer

The `ApexCodeCoverageAnalyzer` is focused specifically on gathering, processing, and summarizing Apex code coverage metrics efficiently.

### Key Features

* **Asynchronous Processing:** Uses batching and concurrency control (`MAX_CONCURRENT_REQUESTS = 5`) to efficiently query the `ApexCodeCoverageAggregate` Tooling API object for large orgs.
* **ID Normalization:** Ensures consistency by normalizing all 18-character Salesforce IDs to the **15-character** case-safe format for internal mapping and querying.
* **Latest Coverage Aggregation:** Correctly aggregates coverage by class ID, ensuring that only the **latest** coverage result (based on `LastModifiedDate`) is used for each component.
* **Detailed Coverage Metrics:** For each class, it calculates:
    * `numLinesCovered`, `numLinesUncovered`, `totalLines`.
    * `coveragePercentage`.
    * `coverageLastModifiedDate` (timestamp of the last test run).
* **Performance and Quality Analysis:** Provides targeted lists for action:
    * Classes with **`poorCoverage`** ($\text{Coverage} < 50\%$).
    * Classes **`belowProductionThreshold`** ($\text{Coverage} < 75\%$).
    * Classes with **`staleCoverage`** ($\text{LastTestDate} > 30 \text{ days}$ ago).

### Output Metrics

| Metric Category | Key Metrics/Findings |
| :--- | :--- |
| **Overall Stats** | Total Classes Analyzed, Classes With/Without Coverage, **Average Coverage Percentage**. |
| **Distribution** | Coverage Distribution: Excellent ($\ge 90\%$), Good ($\ge 75\%$), Fair ($\ge 50\%$), Poor ($< 50\%$). |
| **Performance** | Top Performers ($\ge 90\%$), Needs Improvement ($< 75\%$), Recently Tested, Stale Coverage. |
| **Insights** | Recommendations (e.g., focus on low coverage, re-run stale tests), Warnings (e.g., below 75% average). |
