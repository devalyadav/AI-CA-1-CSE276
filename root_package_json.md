{
  "name": "pantrychef",
  "version": "1.0.0",
  "private": true,
  "description": "An Applied Generative AI Progressive Web App for Zero-Waste Cooking",
  "workspaces": [
    "shared",
    "server",
    "client"
  ],
  "scripts": {
    "dev": "concurrently \"npm run dev --workspace=server\" \"npm run dev --workspace=client\"",
    "build": "npm run build --workspace=shared && npm run build --workspace=client && npm run build --workspace=server",
    "start": "npm run start --workspace=server",
    "test": "npm run test --workspace=server && npm run test --workspace=client",
    "lint": "npm run lint --workspace=shared && npm run lint --workspace=server && npm run lint --workspace=client",
    "check:secrets": "node scripts/check-secrets.mjs",
    "generate:icons": "node scripts/generate-icons.mjs"
  },
  "devDependencies": {
    "concurrently": "^8.2.2",
    "sharp": "^0.33.5",
    "typescript": "^5.6.2"
  },
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  }
}