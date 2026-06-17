# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**deutsch-lernen** is a German language learning web app with a React + TypeScript frontend and a Python FastAPI backend.

## Stack

- **Frontend**: React, TypeScript — lives in `frontend/` (see `frontend/CLAUDE.md`)
- **Backend**: Python, FastAPI — lives in `backend/` (see `backend/CLAUDE.md`)

## Architecture

The app is split into two independently runnable services:

- `frontend/` — SPA served by Vite in dev; talks to the FastAPI server via `VITE_API_URL`
- `backend/` — FastAPI REST API; returns JSON to the frontend

In production the frontend is built and served statically; the FastAPI server handles only `/api/*` routes.
