{
  "name": "@pantrychef/client",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "lint": "tsc --noEmit"
  },
  "dependencies": {
    "@pantrychef/shared": "*",
    "idb-keyval": "^6.2.1",
    "lucide-react": "^0.445.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.5.0",
    "@testing-library/react": "^16.0.1",
    "@types/react": "^18.3.8",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "autoprefixer": "^10.4.20",
    "jsdom": "^25.0.1",
    "postcss": "^8.4.47",
    "tailwindcss": "^3.4.12",
    "typescript": "^5.6.2",
    "vite": "^5.4.8",
    "vite-plugin-pwa": "^0.20.5",
    "vitest": "^2.1.9"
  }
}