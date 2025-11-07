433-betting/
├─ backend/
│  ├─ package.json
│  ├─ index.js                # Express mock API (auth + bets)
│  └─ data.json               # optional persistence file (auto-created)
├─ frontend/
│  ├─ package.json
│  ├─ postcss.config.cjs
│  ├─ tailwind.config.cjs
│  ├─ vite.config.js
│  ├─ src/
│  │  ├─ main.jsx
│  │  ├─ App.jsx
│  │  ├─ index.css
│  │  ├─ services/
│  │  │  ├─ api.js          # fetch wrapper for backend + local fallback
│  │  │  └─ auth.js         # simple auth helpers + localStorage token
│  │  └─ components/
│  │     ├─ Header.jsx
│  │     ├─ MatchesList.jsx
│  │     ├─ BetSlip.jsx
│  │     ├─ Wallet.jsx
│  │     └─ AgeModal.jsx
│  └─ README_frontend.md
└─ README.md{
  "name": "433-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "autoprefixer": "^10.4.14",
    "postcss": "^8.4.24",
    "tailwindcss": "^3.4.8",
    "vite": "^5.0.0"
  }
}module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};module.exports = {
  content: ["./index.html", "./src/**/*.{js,jsx,ts,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
};import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173
  }
});<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>433 — Demo Sportsbook</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>@tailwind base;
@tailwind components;
@tailwind utilities;

/* small accessibility visual for focused elements */
:focus {
  outline: 3px solid rgba(99, 102, 241, 0.4);
  outline-offset: 2px;
}import React from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
import "./index.css";


 // simple client-side auth helpers (demo only)
const TOKEN_KEY = "433_demo_token";
const USER_KEY = "433_demo_user";

export function loginDemo(username = "demo_user") {
  const token = btoa(`${username}:${Date.now()}`); // not secure — demo only
  localStorage.setItem(TOKEN_KEY, token);
  localStorage.setItem(USER_KEY, JSON.stringify({ username }));
  return token;
}

export function logout() {
  localStorage.removeItem(TOKEN_KEY);
  localStorage.removeItem(USER_KEY);
}

export function getToken() {
  return localStorage.getItem(TOKEN_KEY);
}

export function getUser() {
  const raw = localStorage.getItem(USER_KEY);
  return raw ? JSON.parse(raw) : null;
}

export function isLoggedIn() {
  return !!getToken();
}// lightweight fetch wrapper that tries backend then falls back to localStorage.
// Backend expected at http://localhost:4000
const API_BASE = import.meta.env.VITE_API_BASE || "http://localhost:4000";

async function safeFetch(path, options = {}) {
  try {
    const res = await fetch(`${API_BASE}${path}`, options);
    if (!res.ok) throw new Error("backend error");
    return res.json();
  } catch (err) {
    // fallback to localStorage persistence for offline/demo usage
    return localFallback(path, options);
  }
}

function localFallback(path, options) {
  // simple patterns: /bets (GET/POST)
  if (path.startsWith("/bets")) {
    if (options?.method === "POST") {
      const body = JSON.parse(options.body || "{}");
      const existing = JSON.parse(localStorage.getItem("433_bets") || "[]");
      const id = `b_${Date.now()}`;
      const record = { id, ...body, createdAt: new Date().toISOString() };
      const next = [record, ...existing];
      localStorage.setItem("433_bets", JSON.stringify(next));
      return Promise.resolve(record);
    } else {
      const existing = JSON.parse(localStorage.getItem("433_bets") || "[]");
      return Promise.resolve(existing);
    }
  }
  // generic fallback
  return Promise.resolve({ ok: false });
}

export async function fetchBets() {
  return safeFetch("/bets");
}

export async function saveBet(bet, token) {
  return safeFetch("/bets", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
    body: JSON.stringify(bet),
  });
}import React from "react";
import { getUser, isLoggedIn, loginDemo, logout } from "../services/auth";

export default function Header({ theme, toggleTheme, onLoginChange }) {
  const user = getUser();
  return (
    <header className="max-w-6xl mx-auto p-4 flex items-center justify-between">
      <div className="flex items-center gap-3">
        <div
          className="w-12 h-12 rounded-lg flex items-center justify-center bg-gradient-to-br from-indigo-500 to-pink-500 text-white font-bold text-xl"
          aria-hidden
        >
          433
        </div>
        <div>
          <h1 className="text-xl font-extrabold">433 — Sportsbook (Demo)</h1>
          <p className="text-sm text-gray-500">Client-side demo — not for real gambling</p>
        </div>
      </div>

      <div className="flex items-center gap-4">
        <div className="text-right">
          <div className="text-xs text-gray-400">Account</div>
          <div className="font-semibold">{user?.username ?? "Guest"}</div>
        </div>

        <button
          onClick={() => {
            toggleTheme();
          }}
          className="px-3 py-2 bg-gray-200 rounded-lg hover:opacity-90"
          aria-pressed={theme === "dark"}
        >
          {theme === "light" ? "Dark" : "Light"}
        </button>

        {isLoggedIn() ? (
          <button
            onClick={() => {
              logout();
              onLoginChange();
            }}
            className="px-3 py-2 rounded-lg border"
          >
            Log out
          </button>
        ) : (
          <button
            onClick={() => {
              loginDemo("player1");
              onLoginChange();
            }}
            className="px-3 py-2 rounded-lg bg-indigo-600 text-white"
          >
            Log in (demo)
          </button>
        )}
      </div>
    </header>
  );import React from "react";

export default function AgeModal({ open, onVerify, onDeny }) {
  if (!open) return null;
  return (
    <div className="fixed inset-0 bg-black/60 flex items-center justify-center z-50" role="dialog" aria-modal="true" aria-labelledby="age-title">
      <div className="bg-white rounded-xl p-6 w-[420px] text-center">
        <h2 id="age-title" className="text-2xl font-bold mb-2">Age Verification</h2>
        <p className="text-sm text-gray-600 mb-4">You must be 18+ to use this demo sportsbook. This demo does not handle real money.</p>
        <div className="flex gap-3 justify-center">
          <button onClick={onVerify} className="px-4 py-2 rounded-lg bg-indigo-600 text-white">I am 18+</button>
          <button onClick={onDeny} className="px-4 py-2 rounded-lg border">I am under 18</button>
        </div>
      </div>
    </div>
  );
}
}


