OpenMRS 2.1.3 Mirebalais: Oncology Regimen Ordering & Dashboard Support
=======================================================================

- [OpenMRS 2.1.3 Mirebalais: Oncology Regimen Ordering & Dashboard Support](#openmrs-213-mirebalais--oncology-regimen-ordering---dashboard-support)
  * [Purpose](#purpose)
  * [Context](#context)
  * [Assumptions](#assumptions)
- [Functional Requirements](#functional-requirements)
- [Use Cases](#use-cases)
- [Technical Flows](#technical-flows)
  * [DR_UC1 - Select chemotherapy regimen from list](#dr-uc1---select-chemotherapy-regimen-from-list)
  * [DR_UC2 - Submit an order for the selected chemotherapy regimen in DR_UC1 use case](#dr-uc2---submit-an-order-for-the-selected-chemotherapy-regimen-in-dr-uc1-use-case)
- [Proposal](#proposal)
- [Extended Content](#extended-content)
- [Technical TODOs/Open Issues](#technical-todos-open-issues)


Purpose
-------

This document captures options gathered during first week of discussions about how to implement ordering of oncology (chemo specifcally) treatment regimens within OpenMRS Mirebalais distro and provide dashboard analytics about their treatment.


Context
-------

This capability is being added as part of a 3-week on-site engagement between PIH and IBM teams. The goal is to provide a completed set of features for use with Mirebalais distro. 

The scope was focused on doctor-centric workflow during initial consultation/diagnosis of a cancer patient which requires doctor order chemotherapy treatment which is likely to be recurring and re-validated during each cycle visit.

This doctor workflow's use cases are not currently part of Mirebalais distro are:
- Order a chemo treatment from a well-known list of oncology regimens (i.e. templates), this includes ability to adjust drug values before submitting final order.
- Modify chemo treatment - after initial ordering - if needed based on doctor's assessment of patient's reaction to treatment regimen.
- Review patient's historical (dashboard) oncology treatment data (i.e. longitudinal view) when needed.


Assumptions
-----------

- OpenMRS 2.1.3 (or higher) platform w/ Mirebalais distro deployment (i.e. concept dictionary, modules, UI, etc).
- Oncology regimens supported in this effort will (at a minimum) cover the scope of Haiti's chemotherapy treatment forms provided:
    - 5FULeucovorin.pptx
    - AC.pptx
    - CarboTaxol.pptx
    - CHOP.pptx
    - CMF.pptx
    - COP.pptx
    - Cyclophosphamide Single Agent.pptx
    - Doxorubicin20.pptx
    - Doxorubicin60.pptx
    - Paclitaxel80.pptx
    - Paclitaxel175.pptx
    - Paclitaxel175ReChallenge.pptx
- Dictionary dataset (i.e. concepts and related metadata) might (and can) be modified using PIH dataset process.
- Data model could be extended or modified, if needed.


Functional Requirements
=======================

The updated system with oncology treatment ordering capability will leverage existing data model objects and api semantics as much as possible. There is already a substantial coverage of required modeling for supporting oncology regimens. Data concepts such as `OrderSet`, `OrderGroup`, and `Order` provide base for most of the above data interactions.


Use Cases
=========

**Doctor (DR)** use cases (primary data model classes involved are `Patient`, `OrderGroup`, `Order`, `OrderSet`, `OrderSetMember`, `OrderType`, `Encounter`):
- DR_UC1: As a doctor, I want to select and edit a chemotherapy regimen from a pre-configured set available during ordering.
- DR_UC2: As a doctor, I want to submit an order for the selected chemotherapy regimen in DR_UC1 use case.
- DR_UC3: As a doctor, I want to modify chemotherapy regimen *initially ordered* for a patient (i.e. before final order confirmation).
- DR_UC4: As a doctor, I want to modify chemotherapy regimen *previously ordered* for a patient (i.e. order is already in data model)

---

**Adminstrative (AD)** use cases (primary data model classes involved are):
- AD_UC1: As a nurse, I want to capture chemotherapy delivery observations for treatment cycle received on a specific visit.
- AD_UC2: As a nurse, I want to ...

---

**Order Templates (OT)** use cases (primary data model classes involved are `OrderSet`, `OrderSetMember`, `OrderType`):
- OT_UC1: As an oncology clinician, I want to author oncology regimen templates for use in EMR solution.
- OT_UC2: As an oncology clinician, I want to update an existing oncology regimen template currently in use in EMR solution.

---


Technical Flows
===============

- Data Model: 
![](images/muraly-data-model.png)

DR_UC1 - Select chemotherapy regimen from list 
------

- Summary flow:
  1. Query all `OrderSet` templates available, returns array
  2. For each `OrderSet` in array above, query each `OrderSet`, `OrderSetMember`, `Concept`, `OrderType`, `OrderTemplate` detail attributes
  3. Use `OrderSet` indication attribute to group regimen classifications (i.e. "Chemotherapy" vs. "HIV")

- Implementation notes: Data encoded in `OrderSetMember.orderTemplate` is a seralized escaped JSON string that must be decoded in presentation later to understand data. When creating `Order` objects in the next use case, the final JSON string must be serialized similarly containing the updated (if applicable) chemotherapy drugs being ordered by doctor in final initial order.

- Sequence Diagram:  
![](https://www.websequencediagrams.com/files/render?link=ULdAQkpjS3tFmqk8LmqX)

- Data Model References:  
 [Class Diagram](#data-model)  
 [OrderSet object](https://docs.openmrs.org/doc/org/openmrs/OrderSet.html)  
 [OrderSet serialization](https://docs.openmrs.org/doc/serialized-form.html#org.openmrs.OrderSet)  

- Samples:  
  
request: `GET http://humci.pih-emr.org:443/mirebalais/ws/rest/v1/orderset`  
response: HTTP 200 [body](samples/get-ordersets-response.json)  
  
request: `GET https://humci.pih-emr.org:443/mirebalais/ws/rest/v1/orderset/c1c121bf-c660-4435-bdcf-04ac6e99c870`  
response: HTTP 200 [body](samples/get-orderset-chop-response.json)  
  
request: `GET https://humci.pih-emr.org:443/mirebalais/ws/rest/v1/orderset/c1c121bf-c660-4435-bdcf-04ac6e99c870/ordersetmember/2ab90f6b-83c3-4278-8dcb-aa9480f07d01`  
response: HTTP 200 [body](samples/get-ordersetmember-response.json)  
  
  
DR_UC2 - Submit an order for the selected chemotherapy regimen in DR_UC1 use case
------

- Summary flow:
  1. Get the current `Provider` (based on who is logged in, get from the session)
  2. Get the `EncounterRole` (this will be fixed for all orders, so just need a hard coded query) `https://humci.pih-emr.org/mirebalais/ws/rest/v1/encounterrole?q=Ordering%20Provider`
  3. Get the `Encounter` type (fixed again, need a hard coded query) `https://humci.pih-emr.org/mirebalais/ws/rest/v1/encountertype?q=Test%20Order`
  4. Get the current `Location` (get from the session)
  5. Get the `Patient` ID (hopefully this comes from the page/url that we are given)
  6. Construct the `OrderGroup` and `Order` objects and submit as described below

- Implementation notes: Response from the `POST /mirebalais/ws/rest/v1/encounter` request returns the same document submitted but with more of the fields completed, e.g. the UUID for the Encounter which has just been created. encounter role is a description of the type of person who is involved in the encounter (e.g. nurse) For existing drug orders, it looks like this is retreived with a query to `https://humci.pih-emr.org/mirebalais/ws/rest/v1/encounterrole?q=Ordering%20Provider`. Provider looks like a wrapper around person - not sure how to get this.

- Sequence Diagram
![](https://www.websequencediagrams.com/files/render?link=7g-em0gAuPSbCNUyClqT)

- Data Model References:
[Class Diagram](#data-model)  
[OrderSet object](https://docs.openmrs.org/doc/org/openmrs/OrderSet.html)  
[OrderSet serialization](https://docs.openmrs.org/doc/serialized-form.html#org.openmrs.OrderSet)  

- Samples:  
  
request: 
response: 

---

OT_UC1 - Authoring an oncology regimen templates for use in EMR solution
------

- Summary flow:
  1. TBD
  
- Implementation notes: TBD

- Requisites:
  - MacOS (Mavericks or higher) instructions:
    1. install python 2.7 or higher, test that you can run it from command-line (type ```exit()``` to quit python prompt)
       ```
          ibmmacbookmario:utils dearmasm$ python
          Python 2.7.10 (default, Oct  6 2017, 22:29:07) 
          [GCC 4.2.1 Compatible Apple LLVM 9.0.0 (clang-900.0.31)] on darwin
          Type "help", "copyright", "credits" or "license" for more information.
          >>>
    2. install python yaml and requests libraries 
       ```
          sudo easy_install pip
          brew install libyaml
          sudo python -m easy_install pyyaml
          sudo pip install requests
          sudo pip install objdict
          sudo pip install enum
       
    3. test python+yaml lib is working
       ```
          ibmmacbookmario:utils dearmasm$ ls -l test.*
          -rw-r--r--  1 dearmasm  staff  96 Jul 23 09:52 test.py
          -rw-r--r--  1 dearmasm  staff  45 Jul 23 09:13 test.yaml
          ibmmacbookmario:utils dearmasm$ python test.py
          Hello World!
          {'hello': {'go': True, 'world': 123, 'here': 'we'}}

- Sequence Diagram
![]()

- Data Model References:
[Class Diagram](#data-model)  
[OrderSet object](https://docs.openmrs.org/doc/org/openmrs/OrderSet.html)  
[OrderSet serialization](https://docs.openmrs.org/doc/serialized-form.html#org.openmrs.OrderSet)  

- Samples:  
  
request: 
response: 



Proposal
========
Several objects in the data model are missing fields that are necessary for the chemotherapy regimen ordering workflow. These objects and the missing fields are:
- OrderSet
    - Category (Chemotherapy, HIV, etc)
    - Number of cycles typical in the regimen
    - Length of cycles typical in the regimen
- OrderGroup
    - Cycle number
    - Physician notes
    - Reason for Order (Chemotherapy, HIV, etc)
    - Number of cycles in the regimen
    - Length of the cycles in the regimen
    - Previous Order Group
    - Parent Order Group
- DrugOrder
    - Dosing reduction
    - Should the dosing reduction apply to subsequent cycles?
- Maximum lifetime dose
- Drug
    - Units for the maximum daily dose


For OrderSet, the Category field will contain a reference to a Concept which describes what a particular OrderSet is typically used to treat. If an OrderSet is used for multiple situations, the Concept for the OrderSet would be a Concept of type Set, which has the multiple Concepts as members of the Set. OrderSet will also be extended via attributes to hold the attributes of number of cycles and length of the cycles. In order to support the grouping of Premedications, Chemotherapy medications, and Post Medications in a regimen, and OrderSet will be created for each, and the regimen OrderSet will contain references to 

OrderGroup will contain a field called "orderGroupReason" which will be a Concept denoting the reason the OrderGroup was "ordered". Physician notes will be contained in an Observation in the Encounter associated with the OrderGroup, as will information such as the calculated dosage and the actual administered dosage. The fields PreviousOrderGroup and ParentOrderGroup will be added to the OrderGroup object to support linking OrderGroups as well as OrderGroup nesting. OrderGroup nesting will be used to group chemotherapy cycles into Premedications, Chemotherapy medications, and Post medications. OrderGroup will also be extended via attributes to associate the fields of cycle number, the number of total cycles in the regimen at that point in time, and the length of the cycles in the regimen.

For dosing reduction, we will implement a new dosing type to capture the dosing reduction. This will capture the percent of the reduction, as well as if the reduction is temporary (applies to just the medication for this cycle), or if it should apply to subsequent orders of this medication in future cycles.

For maximum lifetime dose, it is typically associated with the drug concept or ingredient, and not a specific formulation. As such, we will associate the maximum lifetime dose to a drug's Concept via ConceptAttributes.

Drug currently has a maximumDailyDose field (which is a double), but no concept of the units associated with it. We are intending to add a doseLimitUnits field to the table which is the Concept referring to the particular units that the max and min daily dose fields use.

We would like to capture the concentration of a drug in a more concrete form than just in the name of the Drug, as we need to fetch that value for calculating the dosage in our application. We will store the concentration value and its units serialized in the strength field of the Drug.

Extended Content
================



Technical TODOs/Open Issues
===========================

