NEO Customer Referral App Requirements Document
Document Version: 1.0

Date: 08-05-2025

Author: Paul Smith

1. Introduction

1.1 Purpose

“NEO Customer Referral App” is an internal web portal that enables employees to track and
manage the NEO Bank’s customer get customer program. It provides:

•  End to end visibility of every referral—from initial code generation to account opening;
•
Insight into program performance through real time metrics and leader boards; and
•  Administrative tooling to tune reward amounts and tier thresholds without IT releases.

1.2 Scope

The document is written for both business stakeholders (product owners, compliance,
operations) and technical teams (developers, DBAs, DevOps). It captures what the system must
do and the data it must persist, leaving implementation details (frameworks, hosting, CI/CD,
etc.) to follow on design phases.

2. System Overview

2.1 System Purpose

The app is designed to manage customer referrals to other customers, from referral to new
customer onboarding. The system integrates with external platforms like Salesforce or SQL (for
customer data). It allows internal users to track referral codes, view all codes associated with
different existing customers, and manage new account openings. The system supports various
user roles, each with specific access permissions tailored to their responsibilities.

2.2 System Context

The system will interface with:

- Salesforce CRM: To manage and retrieve existing customer information.

- SQL Server DB: To fetch data related to customers.

3. Functional Requirements

3.1 Data Model

The data model for the application MUST include exactly the data structures described below.

3.1.1 Referral Code

Id: Unique identifier for the Referral Code.

•
•  Customer ID: Reference to the Customer sharing the referral code.
•
•  Status: The current status of the Referral Code, with possible values:

Issued On: the date the code was generated.

o  Active
o

Inactive

•  Expires On: the date the code expires.

3.1.2 Customer

Id: Unique identifier for the customer.

•
•  Name: The name of the customer.
•  Email: The email address of the customer.
•  Phone: The phone number of the customer.
•  Date Joined: The date the customer was registered in the system.
•  Account Status: The current status of the customer account, with possible values:

o  Active
o

Inactive

3.1.3 Invited Person

Id: Unique identifier for the invited person.

•
•  Name: The name of the person invited.
•  Email: The email address of the person invited.
•  Phone: The phone number of the person invited.
•  Account Status: The current status of the customer account, with possible values:

Invited

o
o  Referral Accepted
o  Referral Rejected
o  Customer

3.1.4 Referral

Invited Person ID: Reference to the Invited Person that was created from the referral.

Id: Unique identifier for the referral.

•
•  Customer ID: Reference to the Customer that created the referral.
•
•  Referral Code ID: Reference to the code that was used for this referral.
•  Campaign ID: Reference to the Campaign associated with this referral.
•  Referral Status: The current status of the Referral, with possible values:

o  Started
o  Approved
o  Rejected
o  Completed

•  Starting Date: Date of process start.
•  Completed Date: Date of process completion.

3.1.5 Reward

•

Id: Unique identifier for the reward.

•  Name: The name of the reward. (For example “Cash Campaign 2025”)
•  Description: text describing the reward.
•  Value: Value in euros.
•  Territory ID: Reference to the Territory associated with the Reward.
•  Reward Status: The current status of the Reward, with possible values:

o  Active
o

Inactive

3.1.6 Territory (IMPORTANT!!)

•
Id: Unique identifier for the Territory.
•  Name: text with the Territory name.
•  Description: text describing the Territory.

3.1.7 Campaign

Id: Unique identifier for the campaign.

•
•  Name: The name of the campaign (For example “Cash Campaign 2025”).
•  Description: text describing the campaign.
•  Reward ID: Reference to the Reward associated with the campaign.
•  Value: Value in euros.
•  Status: The current status of the Campaign, with possible values:

o  Active
o

Inactive

3.2 Main Features

3.2.1 Referral Code Creation

•  Ability to create new referral codes for different accounts.
•
•  Automatic generation of a referral code that accounts can share.

Integration with Salesforce to automatically retrieve and populate customer data.

3.2.2 Reward Management

•  Ability to create and edit rewards;
•  Ability to view all the rewards by status;

3.2.3 Campaign Management

•  Ability to create campaigns and associate them to rewards.
•  Ability to view and edit the campaigns by status.

3.2.4 Referral Creation

•  Ability to create referrals and associate them to different referee and referred accounts.

3.2.5 Referral Editing

•  Ability to modify existing referrals;
•  Ability to change their statuses;

3.2.6 Referral Rewards Management

•  Ability to manage the Referral Rewards statuses and to view them;

4. Roles and Permissions

4.1 Admin

•  Permissions: Full access to all data and features.
•  Capabilities:

o  View, create, edit, delete all data entities.
o  Change entities status.
o  Manage system configurations and user roles.

4.2 Employee

•  Permissions: Access to all data.
•  Capabilities:

o  View, create, and edit all data.
o  Change campaign, rewards and referral status.
o  Access customer information.

5. Non-Functional Requirements

5.1 Performance

•  The system should handle up to 1,000 simultaneous users without performance

degradation.

•  Referrals should be processed and confirmed within 2 seconds on average.

