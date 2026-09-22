# CHEFS Activity Sign-Up App – Development Requirements

## 1. School Alliance Venue Sign-Up Module

### 1.1 Purpose

The School Alliance Venue Sign-Up function is primarily designed to manage daily activity registration at CHEFS school alliance venues, including:

* Home-venue member benefits
* Waiting lists and automatic replacement
* Activity/credit tracking
* Fees for non-alliance members

The system should be as automated as possible to reduce the amount of manual work required by administrators for:

* Tracking participation
* Collecting fees
* Adjusting participant lists
* Deducting activity credits

---

## 2. Creating a Venue Activity

When an administrator creates a school alliance activity, the following information must be configured:

* School / venue name
* Activity date
* Activity start and end time
* Maximum number of primary participants
* Maximum number of waiting-list participants
* School alliance membership associated with the venue
* Fee for a non-alliance member who is successfully promoted from the waiting list
* Registration opening time
* Registration closing time

### Registration Opening Rule

As a general principle, registration should open at least **2 days before the activity**.

For example:

**Activity:**
September 30, 8:00 PM–10:00 PM

The system should open registration on or before September 28.

The exact opening time may be configured by the administrator.

---

## 3. Primary Participant Settings

### 3.1 Participant Limit

Each venue/activity should normally allow a maximum of:

**6 primary participants**

The administrator can modify the maximum number according to the actual capacity of each venue.

### 3.2 Home-Venue School Alliance Members

A school alliance member who belongs to the school/venue hosting the activity may:

* Register as a primary participant
* Participate without paying an additional venue fee
* Be clearly identified by the system as a **Home-Venue School Alliance Member**
* Have **1 activity credit** automatically deducted after completing the activity

---

## 4. Waiting List

When all 6 primary participant positions have been filled, subsequent applicants are automatically placed on the:

**Waiting List**

The maximum number of waiting-list participants is configurable by the administrator.

Waiting-list positions are assigned according to the time at which the registration was successfully completed:

**Waiting #1 → Waiting #2 → Waiting #3 → Waiting #4 → etc.**

In principle, members should **not be allowed to manually change their waiting-list position**.

---

## 5. Cancellation and Automatic Replacement

If a primary participant cancels the activity for personal reasons:

1. The system automatically removes the participant from the primary participant list.
2. The person at the top of the waiting list is automatically promoted to the primary participant list.
3. The remaining waiting-list participants move up one position.

For example:

**Before cancellation:**

* Primary participants: 6 people
* Waiting #1
* Waiting #2
* Waiting #3

If a primary participant cancels:

* Waiting #1 → Primary Participant
* Waiting #2 → Waiting #1
* Waiting #3 → Waiting #2

The system should automatically notify the person who was promoted, for example:

> "You have been automatically promoted from the waiting list to the official participant list for this activity."

---

## 6. Cancellation Rules for Home-Venue School Alliance Members

If a home-venue school alliance member has already registered as a primary participant but later cancels for personal reasons:

* In principle, the club does **not** refund or restore that activity credit.
* However, if another waiting-list participant successfully replaces the member, the administrator may decide whether the original member's activity credit should be restored.

The administrator backend should therefore provide two options:

* **Restore Activity Credit**
* **Do Not Restore Activity Credit**

---

## 7. Members from Other School Alliances Participating at Another Venue

If a member belonging to School Alliance A registers for an activity hosted by School Alliance B:

### While on the Waiting List

No activity credit should be deducted while the member remains on the waiting list.

### After Successful Promotion to the Primary Participant List

Once the member is successfully promoted from the waiting list to the primary participant list, the system should automatically deduct **1 activity credit** from the member's own school alliance activity-credit balance.

The system should record:

* Member's original school alliance
* School/venue actually attended
* Activity date
* Number of credits deducted
* Deduction reason: **Cross-Venue Participation**

### Example

A member belongs to the **Lorne Park School Alliance** and has:

* 10 activity credits remaining

The member successfully gets promoted from the waiting list and participates in an activity hosted by another school.

After the activity is completed:

**10 credits → 9 credits**

---

## 8. Non-School-Alliance Member Participation

Non-school-alliance members may register for the waiting list.

When they enter the waiting list, the system should clearly display:

**Non-School-Alliance Member | Waiting List**

### If They Are Not Promoted

If the member remains on the waiting list and does not ultimately participate:

* No venue fee is charged.

### If They Are Successfully Promoted

If the member is successfully promoted to the primary participant list:

* The system automatically creates a **$5 venue fee**
* The payment status should be displayed as:

  * **Unpaid**
  * **Paid**

The system may support online payment in the future, or the administrator may manually confirm payment.

---

## 9. Automatic Activity Credit Management

The system should create an **Activity Credit Account** for every school alliance member.

The member's account should display:

* Total activity credits
* Used activity credits
* Remaining activity credits

### Example

**Total Credits:** 32
**Used:** 8
**Remaining:** 24

When a member successfully participates in a home-venue activity:

**24 → 23**

When a member successfully participates in an activity at another school/venue:

**24 → 23**

If a member only enters the waiting list but is never promoted and does not participate:

* No activity credit is deducted.

---

## 10. Automatic Processing After an Activity

After each activity ends, the system should process the final participation information.

### Home-Venue School Alliance Member

If the member successfully participated:

* Deduct 1 activity credit.

### Member from Another School Alliance

If the member successfully participated:

* Deduct 1 activity credit from the member's own school alliance activity-credit balance.

### Non-School-Alliance Member

If the member successfully participated:

* Create a **$5 venue fee** record.

### Waiting-List Member Who Was Not Promoted

If the member was not successfully promoted:

* No activity credit is deducted.
* No fee is charged.

---

## 11. Member-Facing App

After logging into the app, members should be able to clearly view:

### My School Alliance

* Affiliated school
* Membership validity period
* Total activity credits
* Used activity credits
* Remaining activity credits

### My Activities

* Registered activities
* Primary participant status
* Waiting-list status and current position
* Activity date and time
* Activity venue
* Registration status
* Whether payment is required
* Payment status

---

## 12. Administrator Backend

Administrators should be able to view and manage:

* All school alliance venues
* Member lists for each venue
* Primary participant lists for each activity
* Waiting lists
* Automatic replacement/promotion records
* Members' remaining activity credits
* Cross-venue participation records
* Non-alliance member participation records
* Outstanding $5 venue fees
* Paid/unpaid records
* Cancellation records
* Exception records

### Administrator Override Permissions

Administrators should have special permissions to:

* Increase or decrease activity credits
* Adjust the official participant list
* Adjust the waiting list
* Waive fees
* Confirm payments
* Restore incorrectly deducted activity credits
* Close or reopen registration

All manual administrative changes should be recorded in an **audit log** for future reference.

---

## 13. Core Automation Logic

The most important logic of the School Alliance Sign-Up System can be summarized as:

**Registration → Determine Member Type → Determine Home Venue / Cross-Venue Status → Determine Availability → Primary List or Waiting List → Automatic Replacement When Someone Cancels → Activity Completion → Automatically Deduct Credits or Charge Fees Based on Final Participation**

### Decision Logic

#### Step 1 – Is the applicant a home-venue school alliance member?

**Yes:**

* Apply home-venue membership benefits.

**No:**

* Proceed to Step 2.

#### Step 2 – Is the applicant a member of another school alliance?

**Yes:**

* If successfully participating, deduct 1 activity credit from the member's own school alliance balance.

**No:**

* Proceed to Step 3.

#### Step 3 – Is the applicant a non-school-alliance member?

**Yes:**

* The applicant may join the waiting list.
* If successfully promoted and participates, charge a **$5 venue fee**.

---

## 14. Recommended Additional Rules and Features

To minimize future disputes regarding activity credits, fees, and participant lists, the following features are strongly recommended.

### 14.1 Check-In Function

The final activity-credit deduction and fee calculation should preferably be based on **actual check-in/attendance**, rather than solely on the registration list.

### 14.2 Cancellation Deadline

The administrator should be able to configure a cancellation deadline, such as:

> Members may not cancel their registration within X hours before the activity starts.

### 14.3 No-Show Records

If a member:

* Is officially registered,
* Does not cancel, and
* Does not attend,

the system should record the member as a:

**No-Show**

### 14.4 Automatic Notifications

The system should automatically send notifications for events such as:

* Successful registration
* Placement on the waiting list
* Successful promotion from the waiting list
* Activity cancellation
* Changes to the activity date/time
* Fee generation
* Other important registration/status changes

### 14.5 Administrative Audit Log

All manual administrative changes should record:

* Administrator/user who made the change
* Date and time of the change
* Information before the change
* Information after the change

This will help minimize disputes regarding:

* Activity credits
* Fees
* Participant lists
* Waiting-list positions
* Cancellations
* Administrative adjustments
