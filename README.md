# Vue 3 + TypeScript + Vite

This template should help get you started developing with Vue 3 and TypeScript in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

Learn more about the recommended Project Setup and IDE Support in the [Vue Docs TypeScript Guide](https://vuejs.org/guide/typescript/overview.html#project-setup).

# Public Toilet Finder


## Overview of Project
The Public Toilet Finder helps users locate public toilet facilities across Germany.
All toilet data is extracted from OpenStreetMap, filtered specifically for the Germany geographic boundary, and served through GeoServer.
The application provides an interactive map, toilet details, user location, and walking-navigation via the Valhalla routing service.

📍 Note:
This version of the Public Toilet Finder is restricted to Germany only and displays toilets exclusively within German boundaries.


## Features of the project 

- Germany-wide public toilet coverage
- WFS toilet data + WMS background using GeoServer
- Interactive OpenLayers map
- Popups with name, access, wheelchair info, fees, opening hours
- User geolocation support
- Walking route generation with Valhalla
- Fully responsive & WCAG-compliant interface
- Built using modern technologies: Vue 3 + TypeScript + Vite


## Project Structure:
/src
  App.vue
  main.ts
  /components
      ToiletMap.vue
  /types
  /assets



## Technology Stack

- Frontend: Vue.js, TypeScript, OpenLayers, Vite
- Backend / Data :GeoServer (WMS + WFS), OpenStreetMap (OSM)
- Routing: Valhalla Routing API
- Tools : Git, ESLint
- Accessibility Testing : Lighthouse, Screen readers



## Data Workflow:

- Extract German toilet data (amenity=toilets) from OpenStreetMap

- Import into GeoServer under a Germany workspace/layer

- Publish through WFS and WMS

- Display using OpenLayers in Vue frontend

- Decode navigation polylines from Valhalla routing


## Installation & Run

1️⃣ Clone the Repository:
git clone <repository-url>
cd public-toilet-finder

2️⃣ Install Dependencies:
npm install

3️⃣ Run Development Server:
npm run dev

Open in browser:
👉 http://localhost:5173/
