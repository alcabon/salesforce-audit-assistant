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

Yes, here is the documentation for the remaining core analysis files in the Salesforce Metadata Analyzer Suite.

***

## 5. SecurityAnalyzer (Basic Health Check) 🛡️

The `SecurityAnalyzer` class provides the foundational **Salesforce Health Check** analysis, focusing on organizational security settings as defined by Salesforce's standard security baselines. This serves as the "basic" security audit level, often required before running the more comprehensive `EnhancedSecurityAnalyzer`.

### Core Functionality

* **Health Check Reproduction:** The analyzer reproduces checks across high-risk, medium-risk, and low-risk security settings by querying Salesforce Metadata, primarily the **`SecuritySettings`** and **`FileUploadAndDownloadSecuritySettings`** types.
* **Settings Covered:**
    * **High-Risk:** Session domain locking, SMS identity activation, Clickjack protection variants, CSRF protection, and HttpOnly attribute requirements.
    * **Medium-Risk:** Minimum password lifetime, force relogin after admin access, continuous IP range enforcement, CSP for email, and content sniffing protection.
    * **Low-Risk:** Password answer obscuring, forced logout on timeout, and session timeout duration.
* **Compliance Status:** Assesses each setting and assigns a status: **`COMPLIANT`**, **`WARNING`**, or **`CRITICAL`**.
* **Output Scoring:** Calculates an **`overallScore`** (0-100) based on weighted compliance, penalizing critical and warning findings.

### Key Output Fields (`HealthCheckResult`)

| Field | Description |
| :--- | :--- |
| **`category`** | High-Risk, Medium-Risk, Low-Risk, or Informational. |
| **`setting`** | The name of the specific security setting being checked (e.g., 'Lock sessions to the domain...'). |
| **`status`** | The compliance result: `COMPLIANT`, `WARNING`, or `CRITICAL`. |
| **`recommendation`** | Actionable steps to remediate the non-compliant setting. |
| **`impact`** | Description of the business/security risk if the issue remains unresolved. |

***

## 6. EnhancedUsageAnalyzer (Dependency Analysis) 🔗

The `EnhancedUsageAnalyzer` is the dedicated engine for performing **metadata dependency analysis**, forming the backbone of the usage reports produced by `ConsistentDependencyAnalyzer` and `EnhancedFieldsAnalyzer`. It uses the Salesforce **Tooling API** to query the **`MetadataComponentDependency`** object efficiently.

### Core Functionality and Optimization

* **Dependency Source:** Queries `MetadataComponentDependency` to identify all components that are **referenced by** or **reference** a target component (e.g., an Apex Class or Custom Field).
* **High-Volume Processing:** Designed to handle large volumes of component IDs by implementing several optimizations:
    * **Batching:** Splits target IDs into chunks that respect the SOQL `IN` clause limit (100 IDs).
    * **Composite Requests:** Packages multiple SOQL queries into **Composite REST API** requests (`MAX_COMPOSITE_REQUEST_SIZE = 5`) to minimize round trips and improve query performance.
    * **Super-Batching:** Processes the Composite Requests in larger groups (`MAX_DEPENDENCY_BATCH_SIZE = 500`) to further manage concurrency and system load.
* **ID Normalization:** Ensures data integrity by **normalizing all 18-character Salesforce IDs to 15-character** case-safe IDs for consistent lookups and processing.
* **Bidirectional Relationship Tracking:** Accurately tracks components that are **doing the referencing** (`MetadataComponentId`) and those that are **being referenced** (`RefMetadataComponentId`), ensuring comprehensive dependency coverage.

### Key Output Fields (`DependencyRecord`)

| Field | Description |
| :--- | :--- |
| **`id15`** | The 15-character ID of the analyzed component (the source component). |
| **`isUsed`** | Boolean indicating if any dependency relationship was found for the component. |
| **`usageCount`** | The total number of components that reference this component. |
| **`dependencies`** | An array of `DependencyDetail` objects, listing every referencing component (name, type, ID). |

***

## 7. EnhancedFieldsAnalyzer (Metadata Only) 🏷️

The `EnhancedFieldsAnalyzer` is an advanced metadata retriever focused solely on gathering and summarizing rich, detailed information about custom fields and standard fields without requiring actual data access. It is the precursor to the data-integrated version, `EnhancedFieldsAnalyzerWithData`.

### Core Functionality

* **Deep Field Metadata:** Queries the `FieldDefinition` Tooling API object to retrieve comprehensive information, including: `DurableId`, security classification, tracking, filtering, indexing, and various relationship details.
* **Field Data for Dependency:** Its primary role is to fetch the necessary field data (specifically the `DurableId` to extract the component ID) needed for the `ConsistentDependencyAnalyzer` to run its usage checks.
* **Batch Processing:** Uses object name batching to efficiently retrieve field metadata across many objects, mitigating SOQL limits (inherits this structure from the full implementation).
* **Initial Analysis:** Performs basic analysis without dependency or data context:
    * **Security Analysis:** Identifies **`unclassifiedFields`** and **`highSecurityFields`** based on metadata tags.
    * **Performance Analysis:** Flags fields with potential impact, such as **`calculatedFields`**, **`largeTextFields`**, and fields recommended for or requiring indexing.

### Key Output Structures

| Structure | Focus |
| :--- | :--- |
| **`EnhancedFieldInfo`** | The core data structure containing all metadata, including `durableId`, `isIndexed`, `isRequired`, and `securityClassification`. |
| **`EnhancedFieldsSummary`** | High-level overview, reporting total fields, custom fields, and counts of initial findings (critical, performance, security). |
| **`FieldUsageStats`** | Placeholder for dependency results, containing only total fields and object breakdowns until integrated with `ConsistentDependencyAnalyzer`. |

***

## 8. ApexCodeCoverageAnalyzer (Code Coverage) 📊

This file has been integrated and described under the **ConsistentDependencyAnalyzer** documentation, as it provides the dedicated engine for that tool.

### Summary of Role

The `ApexCodeCoverageAnalyzer` is responsible for querying, normalizing, and calculating **accurate line-by-line test coverage percentages** for Apex classes using **`ApexCodeCoverageAggregate`**. It handles batching, concurrency, and ID normalization to ensure efficient and consistent reporting of code coverage data.

L'analyse des fichiers a été complétée. Voici la documentation des deux derniers composants fondamentaux de la suite d'analyse de métadonnées Salesforce : le gestionnaire de connexion centralisé et le service d'audit de haut niveau.

***

## 9. SalesforceConnectionManager (Gestion de la Connexion) 🔌

La classe `SalesforceConnectionManager` sert de point de contrôle unique et centralisé pour gérer toutes les connexions Salesforce (via **JSForce**) au sein de la suite d'analyse. Elle élimine la duplication du code de connexion et garantit une gestion cohérente de l'authentification et de l'état de la connexion.

### Fonctionnalités Clés

* **Gestion Centralisée de JSForce** : Crée, gère et valide les connexions JSForce en utilisant l'authentification basée sur **SF CLI** (`sf org display`) pour récupérer les jetons d'accès et les URL d'instance.
* **Mise en Cache des Connexions (Caching)** : Met en cache les connexions établies pour différents alias d'organisation afin d'éviter les reconnexions inutiles et d'améliorer la performance.
* **Validation de la Connexion** : Valide l'état d'une connexion en effectuant un appel à l'API d'identité (Identity API) lors de la connexion initiale et de la réutilisation à partir du cache.
* **Gestion des Événements** : Implémente un modèle d'écouteur/d'événement pour notifier les autres composants (comme `RefactoredAuditService`) des changements d'état de la connexion (établissement, échec, changement d'organisation).
* **Liste des Organisations Disponibles** : Utilise la commande `sf org list --json` pour fournir une liste à jour des alias d'organisation configurés localement.

### Avantages

| Avantage | Description |
| :--- | :--- |
| **Cohérence** | Tous les analyseurs utilisent la même source de connexion, garantissant que l'analyse s'exécute sur l'organisation actuellement active. |
| **Efficacité** | Le cache des connexions et la validation évitent les appels d'authentification répétés et coûteux. |
| **Découplage** | Les services d'analyse n'ont plus besoin de gérer l'authentification eux-mêmes, simplifiant leur architecture. |

---

## 10. RefactoredAuditService (Service d'Audit de Haut Niveau) 🏛️

La classe `RefactoredAuditService` est le **service d'orchestration** de haut niveau qui consolide et exécute les différents types d'audit (Sécurité, Apex, Champs) en utilisant les analyseurs et les gestionnaires de connexion sous-jacents. Elle est conçue pour remplacer plusieurs services d'audit spécialisés par une interface unique et complète.

### Fonctionnalités Clés

* **Orchestration d'Audit** : Fournit des méthodes pour exécuter trois principaux types d'audit :
    * **`runSecurityAudit`** : Utilise `UnifiedSecurityAnalyzer` (ou `EnhancedSecurityAnalyzer`) pour l'audit de sécurité (modes `basic`, `enhanced`, `comprehensive`).
    * **`runApexAudit`** : Utilise `ConsistentDependencyAnalyzer` pour l'analyse des dépendances Apex, de la couverture et de la qualité des tests.
    * **`runFieldsAudit`** : Utilise `EnhancedFieldsAnalyzer` et `ConsistentDependencyAnalyzer` pour l'analyse de l'usage et des dépendances des champs personnalisés.
* **Audit Complet (`runComprehensiveAudit`)** : Exécute tous les types d'audit (`Security`, `Apex`, `Fields`) en **parallèle** pour une efficacité maximale.
* **Amélioration par IA (AI Enhancement)** : Intègre un **`VSCodeLMProvider`** pour générer des analyses, des recommandations et des résumés exécutifs (complets) basés sur l'IA, si l'option est activée et qu'un modèle est spécifié.
* **Gestion des Résultats** :
    * Capture le temps d'exécution (`executionTime`) et les métadonnées de l'analyse.
    * Gère la transformation des données brutes en structures adaptées au tableau de bord (`dashboardData`) et au rapport (`reportData`).
    * Implémente une fonction pour **sauvegarder les résultats d'audit** au format JSON dans un répertoire de l'espace de travail VS Code.
* **Interface unifiée** : Agit comme un écouteur de connexion (`ConnectionEventListener`) pour le `SalesforceConnectionManager`, lui permettant de réagir aux changements d'organisation et aux échecs de connexion.

### Structures de Sortie Clés (`AuditResult`)

| Champ | Description |
| :--- | :--- |
| **`success`** | `true` si l'audit s'est terminé sans erreur fatale, `false` sinon. |
| **`orgAlias`** | L'alias de l'organisation sur laquelle l'audit a été exécuté. |
| **`auditType`** | Le type d'analyse exécuté (e.g., `apex`, `enhanced-security`). |
| **`reportData`** | Les données complètes et structurées du rapport, y compris les résumés détaillés et l'analyse IA (si incluse). |
| **`metadata`** | Métadonnées de l'exécution, incluant le temps d'exécution, l'horodatage, et l'ID du modèle IA utilisé. |

Cette architecture garantit que l'analyse des métadonnées est **modulaire** (les analyseurs font le travail), **performante** (le gestionnaire de connexion et les requêtes par lots gèrent l'API), et **utilisable** (le service d'audit fournit l'interface utilisateur logique et la génération de rapports).
