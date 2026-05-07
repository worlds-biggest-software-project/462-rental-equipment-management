# Rental Equipment Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for equipment rental businesses to track assets, manage bookings, schedule maintenance, and handle customer billing -- replacing fragmented legacy tools with a single, modern system.

Rental Equipment Management is a unified platform for equipment rental operators -- from independent tool-hire shops to mid-size construction fleet companies -- who need to coordinate asset tracking, availability, reservations, inspections, maintenance, and invoicing in one place. The core problem it solves is the disconnect between what is physically available, what is booked, and what is billed, a gap that today causes double-bookings, under-maintained equipment, and billing disputes that erode margins.

---

## Why Rental Equipment Management?

- **Enterprise platforms are priced out of reach for most operators.** Point of Rental and Wynne Systems serve large multi-depot chains but carry enterprise pricing and implementation timelines that exclude the majority of rental businesses with 50-500 assets.
- **Mid-market tools lack GPS telematics and maintenance depth.** Quipli and Booqable offer strong online booking but provide only basic preventive maintenance scheduling and limited or no built-in telematics integration for real-time asset location and hour-meter tracking.
- **No open-source alternative exists.** Every significant player (Quipli, EZRentOut, Point of Rental, Rentman, Booqable) is proprietary SaaS with no open-source components, leaving operators locked into vendor pricing with no self-hosted option.
- **Billing complexity is poorly handled at the mid-market tier.** Custom rate tables covering weekends, holidays, shift arrangements, and minimum charges require substantial configuration effort in tools like EZRentOut, while smaller platforms lack these capabilities entirely.
- **Damage claim workflows remain manual and disconnected.** Linking inspection photos to invoices and insurance claim submissions is an underserved area across all current solutions, creating audit-trail gaps and billing disputes.

---

## Key Features

### Asset Registry and Tracking

- Equipment profiles with serial numbers, purchase cost, current value, and category
- GPS telematics integration (Geotab, Samsara) for real-time asset location and hour-meter data
- Multi-depot inventory with inter-depot transfer management and in-transit visibility
- Certificate-of-insurance tracking with automatic expiry alerts before rental commencement

### Booking and Rental Contracts

- Online booking portal with real-time availability calendar and customer self-service reservation
- Quote-to-contract workflow with digital signature capture and terms enforcement
- Extension and early-return handling within the contract lifecycle
- Reservation conflict detection across all locations with inter-depot transfer fulfilment

### Inspection and Maintenance

- Pre- and post-rental condition checklists with photo attachment
- Automatic damage-charge generation linked to inspection evidence
- Calendar- and meter-based preventive maintenance scheduling with work-order creation
- Out-of-service blocking during maintenance periods to prevent bookings on unavailable equipment

### Invoicing and Payments

- Time-based billing (hourly, daily, weekly) with extras, fuel charges, and deposit management
- Advanced billing engine supporting custom rate tables, minimum charges, and shift pricing
- Online payment via Stripe with QuickBooks and Xero accounting integration
- Automated customer communication: confirmation, reminder, and overdue-return notifications

### Analytics and Optimisation

- Fleet utilisation rates: on-rent, available, and in-service equipment by type and location
- Revenue per asset and maintenance cost vs. rental revenue reporting
- Overdue-return tracking and prediction based on customer history and rental duration patterns
- Demand forecasting by equipment type, season, and region for fleet planning decisions

---

## AI-Native Advantage

AI capabilities differentiate this platform from incumbents in several concrete ways. Demand forecasting predicts equipment utilisation by type, season, and region, enabling data-driven fleet planning and acquisition decisions rather than gut-feel purchasing. Automated damage assessment uses photo comparison of pre- and post-rental condition images to generate charge recommendations, reducing manual inspection overhead and billing disputes. Dynamic pricing adjusts rental rates based on real-time fleet utilisation, local demand, and competitor pricing signals. Overdue return prediction identifies high-risk rentals based on customer history and duration patterns, allowing proactive follow-up before assets go missing.

---

## Tech Stack & Deployment

The platform targets self-hosted and cloud deployment modes, serving operators who need on-premise control as well as those preferring managed hosting. GPS telematics data ingestion requires API integration with providers such as Geotab and Samsara, as well as OEM gateways (Caterpillar, Deere, Komatsu). Accounting sync targets QuickBooks, Xero, and Sage through their standard developer APIs. Payment processing uses Stripe under standard commercial terms. Digital signature capture for rental contracts must comply with jurisdiction-specific e-signature laws (ESIGN Act in the US, eIDAS in Europe). Mobile-first inspection workflows require offline-capable apps for field staff operating in areas with poor connectivity.

---

## Market Context

The global construction equipment rental market was valued at approximately USD 132 billion in 2025 and is forecast to exceed USD 229 billion by 2034 (research.md, citing industry sources). Incumbent SaaS pricing ranges from free tiers for very small operators to USD 500+ per month for full enterprise deployments. The primary buyers are independent and mid-size equipment rental operators with 50-500 assets who are currently underserved by both the enterprise platforms (too expensive, too complex) and the small-business tools (too limited in telematics and maintenance depth).

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
