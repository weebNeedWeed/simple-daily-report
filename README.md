# Simple Daily Report App

[![Vue 3](https://img.shields.io/badge/Vue.js-3.x-4fc08d.svg)](https://vuejs.org/)
[![Vite](https://img.shields.io/badge/Vite-Latest-646cff.svg)](https://vitejs.dev/)
[![Style](https://img.shields.io/badge/Style-Minimalism-E36A6A.svg)](#)

A professional, minimalist productivity application designed to streamline the daily work reporting workflow. Built with performance and user experience as core priorities.

---

## 📖 Table of Contents
- [Project Philosophy](#-project-philosophy)
- [Key Features](#-key-features)
- [Technical Stack](#-technical-built-with)
- [Installation & Setup](#-getting-started)
- [Project Structure](#-project-structure)
- [Future Roadmap](#-future-roadmap)

---

## 🧠 Project Philosophy
Reporting shouldn't be a chore. This app follows the **Minimalism** principle: reducing cognitive load by removing unnecessary UI elements, focusing on high-quality typography, and providing smart keyboard-driven shortcuts.

## ✨ Key Features

### 🖋️ Distraction-Free Editing
- **Focus Mode (Zoom)**: Toggle full-screen mode for individual report sections to concentrate on lengthy descriptions.
- **Smart Bullet Rotation**: Intelligent list management using `Tab` and `Shift + Tab` to rotate through professional bullet styles (`-`, `+`, `*`, `**`).
- **Auto-continuation**: Pressing `Enter` automatically maintains your current list level or cleanly exits on an empty line.

### 💾 Robust Persistence
- **Debounced Auto-save**: State changes are automatically persisted to `localStorage` 1000ms after user activity ceases.
- **Save State Indicators**: Real-time visual feedback on whether your work is "Saved", "Saving", or "Unsaved".

### 📊 Comprehensive History
- **Temporal Navigation**: A persistent sidebar allows users to jump between dates, reviewing or updating previous reports instantly.
- **Local Database**: All data is stored locally in the browser, ensuring privacy and instant access without backend latency.

### 📤 Smart Export
- **Multi-Format Copy**: Choose between copying the "Completed Tasks Only" for quick status updates or the "Full Report" for official records.
- **Formatting Preservation**: Preserves nested indentation and spacing for perfect rendering in Slack, Discord, or Zalo.

---

## 🚀 Getting Started

### Prerequisites
- [Node.js](https://nodejs.org/) (Version 20.19.0+ or 22.12.0+)
- [npm](https://www.npmjs.com/) or [yarn](https://yarnpkg.com/)

### Installation
```bash
# Clone the repository
git clone <repository-url>

# Navigate to the directory
cd daily-report-app

# Install dependencies
npm install
```

### Development
```bash
# Spin up the development server with Hot Module Replacement (HMR)
npm run dev
```

### Production
```bash
# Build the optimized production bundle
npm run build
```

---

## 🛠 Built With
- **Logic**: [Vue 3](https://vuejs.org/) (Composition API)
- **Tooling**: [Vite](https://vitejs.dev/)
- **Styling**: Vanilla CSS with Design Tokens
- **Typography**: [Be Vietnam Pro](https://fonts.google.com/specimen/Be+Vietnam+Pro)
- **Storage**: Browser LocalStorage API

---

## 📁 Project Structure
```text
src/
├── assets/             # Global styles and base design tokens
├── components/         # Modular Vue components (DailyForm, History)
├── App.vue             # Main layout and state orchestration
└── main.js             # Application entry point
```

## 📅 Future Roadmap
- [ ] Rich text editor (Markdown) support.
- [ ] Export to PDF/Markdown file.
- [ ] Dark Mode toggle.
- [ ] Weekly/Monthly summary generation.

---
**Crafted for developers who value speed and aesthetics.**
