# Zenspace App Guide

## What This App Does

Zenspace helps teams manage spaces that can be booked by internal users or external customers.

The app has two sides:

- The admin side, where operators set up and manage the business
- The booking side, where customers or end users browse and reserve spaces

In simple terms, the app supports this full journey:

1. Set up an organization
2. Add groups or locations
3. Create meeting spaces inside those groups
4. Configure pricing, availability, devices, vouchers, and rules
5. Let customers book spaces through the booking app
6. Manage bookings and operations from the admin dashboard

## Who Uses It

### Admins and Operations Teams

They use the dashboard to:

- create and organize groups
- add meeting spaces
- manage bookings
- set prices and discounts
- connect devices and integrations
- configure rules, notifications, and settings

### Customers or End Users

They use the booking experience to:

- browse available spaces
- pick a date and time
- review price
- complete payment
- see booking confirmation

## Main User Flows

## 1. Sign In and Enter the Workspace

The first step is signing in.

After authentication, the user enters the organization workspace flow. If the user belongs to multiple organizations, they can choose which workspace to enter.

Typical flow:

1. Open the app
2. Sign in
3. Select an organization
4. Enter the dashboard

Why this matters:

- Everything in the dashboard is organization-based
- Users only see the data and settings for the organization they are working in

## 2. Create or Select an Organization

Organizations are the top-level business container in the app.

An organization usually represents a company, brand, or operating business using Zenspace.

Inside an organization, teams can manage:

- groups
- spaces
- users
- bookings
- pricing
- settings

Typical flow:

1. Open the organization list
2. Search or choose an organization
3. Or create a new organization
4. Enter that organization’s workspace

## 3. Set Up Groups or Locations

Groups represent the next level under an organization.

A group often acts like a location, branch, building, or collection of spaces.

A group usually contains:

- name and identity
- address and timezone
- images and public-facing presentation
- meeting spaces that belong to that group

Typical flow:

1. Open the groups page
2. Create a group
3. Add address and timezone
4. Add images or presentation content
5. Start adding meeting spaces inside that group

Why groups are important:

- Booking and availability often depend on group context
- The booking app can expose a group-level public experience

## 4. Create and Manage Meeting Spaces

Meeting spaces are the main bookable units in the platform.

A meeting space can represent:

- a meeting room
- a desk area
- a studio
- a private office
- another reservable space

Admins can configure:

- name and code
- room details and description
- capacity
- images and thumbnails
- booking duration rules
- availability window
- base price and deposit-related values
- active or inactive state

Typical flow:

1. Open a group
2. Create a meeting space
3. Fill in basic details
4. Add booking rules
5. Add price details
6. Publish or activate the space

After a meeting space is created, admins can go deeper into:

- bookings for that space
- dynamic pricing
- devices
- physical-space mapping
- space unavailability
- logs and integrations

## 5. Manage Availability

Availability controls when a space can be booked.

This usually depends on several layers working together:

- the meeting space availability window
- business hours
- blocked or unavailable ranges
- current bookings
- pricing or booking rules

Typical flow:

1. Set the allowed booking date window
2. Define business hours
3. Add unavailable periods when needed
4. Review how booking slots appear in the booking app

This helps teams prevent bookings outside of valid operating hours.

## 6. Configure Pricing

Pricing in Zenspace starts with a base price on the meeting space, then can be adjusted with dynamic pricing rules.

Pricing can be influenced by:

- fixed values
- multipliers
- adjustments
- date-based rules
- time-based rules

Typical flow:

1. Set the base meeting space price
2. Open dynamic pricing
3. Create a rule
4. Choose when the rule applies
5. Choose how the rule changes the price
6. Save and test the outcome in the booking flow

This allows teams to charge differently for:

- peak hours
- weekends
- date ranges
- special events

## 7. Create Vouchers and Discounts

Vouchers let admins create promotional or controlled discount campaigns.

They can be used to:

- attract new bookings
- reward selected users
- run limited campaigns
- reduce price for certain conditions

Typical flow:

1. Open vouchers
2. Create a voucher
3. Set the code and discount type
4. Add date limits or usage limits
5. Restrict where or when it can be used
6. Publish the voucher

During booking, valid vouchers can reduce the booking amount before payment.

## 8. Manage Bookings

Bookings are one of the most important operational parts of the app.

The bookings area helps teams:

- review all reservations
- check status and payment info
- inspect customer details
- cancel bookings when needed
- understand what is happening across spaces

Typical flow:

1. Open the bookings page
2. Filter or search bookings
3. Open a booking
4. Review status, customer, and payment details
5. Cancel or update when needed

This is where day-to-day operators usually spend a lot of time.

## 9. Connect Devices and Operational Access

Some spaces are connected to devices such as locks, access systems, or other hardware.

The app includes areas for:

- devices
- device logs
- user-device access

Typical flow:

1. Add or view devices linked to a space
2. Review device status
3. Inspect logs for operational issues
4. Manage which users or bookings get access

This helps bridge the booking system with the physical world.

## 10. Connect Physical Spaces to Virtual Spaces

Some teams use a physical-to-virtual mapping flow where the physical space exists in another system, and Zenspace stores the mapping relationship.

This flow lets admins:

- connect a Zenspace meeting space to a physical space
- define the date window for that connection
- review mapping conflicts
- unlink existing mappings if needed

Typical flow:

1. Open a meeting space
2. Go to the physical space tab
3. Connect to physical spaces
4. Choose a physical space
5. Set the mapping period
6. Save the mapping

There is a dedicated guide for this flow:

- `docs/PHYSICAL_SPACE_MAPPING_GUIDE.md`

## 11. Configure Integrations and Notifications

Zenspace includes operational tools beyond basic bookings.

These include:

- integrations
- webhooks
- notifications
- device failure policy
- API keys

Typical flow:

1. Open the relevant settings or integration page
2. Configure the connection or rule
3. Save the configuration
4. Monitor the resulting behavior

These areas are especially useful for teams integrating Zenspace with external platforms or internal workflows.

## 12. Use the Public Booking App

The booking app is the customer-facing side of the product.

A customer usually sees:

- available spaces
- date and time pickers
- pricing summary
- booking form
- payment step
- confirmation page

Typical customer flow:

1. Open the booking page for a group or space
2. Choose a date
3. Choose a time range
4. Review price
5. Enter booking details
6. Apply voucher if available
7. Pay
8. See booking confirmation

This flow is powered by the same admin-managed data:

- groups
- meeting spaces
- pricing rules
- business hours
- availability
- vouchers

## 13. Common Day-to-Day Admin Journeys

Here are the most common real-world journeys in the app.

### Launching a New Space

1. Create or select the organization
2. Create a group or location
3. Add a meeting space
4. Upload images
5. Set booking rules and price
6. Configure dynamic pricing if needed
7. Activate the space
8. Test the public booking page

### Handling a Booking Problem

1. Open bookings
2. Search for the customer or booking
3. Review status and payment info
4. Inspect the meeting space and availability
5. Cancel or adjust the booking if necessary

### Running a Promotion

1. Create a voucher
2. Define usage rules
3. Limit by date or scope if needed
4. Publish the code
5. Monitor booking usage

### Troubleshooting Space Operations

1. Open the meeting space
2. Review linked devices
3. Inspect device logs
4. Review booking access or physical mapping
5. Adjust operational settings

## How the App Is Organized for Users

From a user perspective, the app is easiest to understand in this order:

1. Organization
2. Group
3. Meeting space
4. Availability and pricing
5. Booking experience
6. Operations and integrations

That means:

- organizations contain groups
- groups contain meeting spaces
- meeting spaces are what customers book
- bookings depend on availability, price, and rules
- devices and integrations support real-world operations

## Related Docs

For deeper feature-specific details, see:

- `docs/PHYSICAL_SPACE_MAPPING_GUIDE.md`
- `docs/booking-iframe-embed.md`
- `docs/MEETING_SPACE_AVAILABILITY_IMPLEMENTATION_PLAN.md`
- `docs/GROUP_AVAILABILITY_IMPLEMENTATION_PLAN.md`
- `docs/VOUCHER_IMPLEMENTATION_PLAN.md`
- `docs/IMAGE_SPECIFICATIONS.md`

## Summary

Zenspace is a complete space operations and booking platform.

Admins use it to build and manage the workspace structure, configure pricing and availability, connect operational systems, and monitor bookings.

Customers use it to discover spaces, choose a suitable time, pay, and complete reservations.

If someone is new to the product, the easiest way to learn it is:

1. Start with organization and group setup
2. Learn how meeting spaces are configured
3. Understand how pricing and availability affect booking
4. Follow the customer booking flow from start to finish
5. Use bookings, devices, and settings for daily operations
