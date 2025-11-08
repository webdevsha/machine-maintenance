# Machine Lifetime Widget

This is a self-contained, single-file HTML/JavaScript widget designed to track and visualize the operational lifetime of machines. It plots a machine's baseline 10-year lifespan and then calculates an extended lifespan based on maintenance logs.

The entire application runs in the browser and uses localStorage for data persistence, meaning all user accounts and machine data are stored locally on that specific browser.

## 🚀 How to Use

### Directly in Browser:
1. Save the file as `index.html`
2. Open it in any modern web browser

### In WordPress:
1. Open a Page or Post editor
2. Add a "Custom HTML" block
3. Paste the entire content of the HTML file into the block
4. Save and view the page

## ✨ Features

- **Local User System**: A simple login/register/demo system that stores "users" in localStorage
- **Machine Management**: Add, save, and select multiple machines, each with a name, unique ID, and purchase date
- **Maintenance Logging**: Log maintenance events with details like date, person-in-charge, role (Student, Staff, Technician), and notes
- **Two-Tier Maintenance**: Supports two types of maintenance with different effects:
  - `kendiri` (Self/Minor)
  - `major` (Major)
- **Lifetime Visualization**:
  - Uses Chart.js to plot the machine's lifetime
  - Baseline: A 10-year linear degradation from 100% to 0%
  - Adjusted Lifetime: A new curve showing the lifetime extended by maintenance
  - Event Markers: Plots scatter points on the chart for each kendiri or major maintenance event
- **Lifetime Calculation**: Displays the current baseline percentage, the adjusted percentage, and the new extended End-of-Life date
- **Dummy Data Generator**: Includes a button to auto-generate a history of maintenance logs for testing

## ⚙️ Core Logic

The widget operates on a few key business rules defined in the JavaScript:

- **Baseline Lifetime**: Every machine is assumed to have a 10-year (3650 days) lifespan from its purchase date, degrading linearly
- **Maintenance Effects**: Each maintenance log adds a permanent percentage boost to the machine's lifetime value:
  - `kendiri` (self): +0.01%
  - `major` (major): +1.0%
- **Lifetime Extension**: The total percentage boost from all maintenance events directly extends the 10-year EOL date
  - Formula: `YearsExtended = TotalBoostPercent / 10`
  - Example: 10 "major" maintenances = +10% boost = 1 extra year of life
- **Visual Smoothing**: To make the chart look more realistic (avoiding sharp vertical jumps), the positive effect of each maintenance event is visually smoothed in over a 6-month period on the graph

## 🔌 Dependencies

- **Chart.js (v4.3.0)**: Loaded via CDN for all charting

## ⚠️ Note for Production Use

This widget is a demo/prototype. The use of localStorage means:

- Data is not secure
- Data cannot be shared between different users, computers, or browsers
- Clearing browser data will delete all users and machine logs

For a multi-user, production-ready version, the storage functions (save, load) must be rewritten to communicate with a proper backend database (e.g., via a WordPress REST API, Firebase, or a custom API).

## About

CMM/SMAW Machine Maintenance

### Resources
- Readme
- Activity

### Stats
- Stars: 0 stars
- Watchers: 0 watching
- Forks: 0 forks

### Releases
No releases published  
*Create a new release*

### Packages
No packages published  
*Publish your first package*

### Deployments (7)
- github-pages (21 minutes ago)
- *+ 6 deployments*

### Languages
- HTML: 100.0%

---

**Footer**  
© 2025 GitHub, Inc.

*Footer navigation: Terms | Privacy | Security*
