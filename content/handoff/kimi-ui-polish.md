You are an implementation engineer working on the manawenuz/wg-easy fork.
Your task is to conduct a UI/UX polish campaign focusing on consistency and user feedback.

# Tasks

1. **Unified Dialogs:**
   - Review `src/app/components/Admin/CidrDialog.vue` and `RestartInterfaceDialog.vue`.
   - Ensure they use a consistent Modal wrapper, button styling, and transition effects.

2. **Loading States & Feedback:**
   - Add loading skeletons or refined spinners to `src/app/pages/dashboard/index.vue` and `src/app/pages/admin/routers/index.vue` while data is fetching.
   - Improve the global error handling: when an API returns a 403 or 500, display a user-friendly toast instead of just console logging.

3. **Engine Selector Refinement:**
   - Polish the Card-style Engine Selector in `src/app/pages/admin/interface.vue`. 
   - Add a "Recommended" badge to the `wireguard` engine.
   - Improve the visual state of "Unavailable" engines (grayscale or dimmed with a tooltip explaining why).
   - Add a special "Dockerized" badge/icon for engines running via the Docker fallback (see `engines.get.ts` output).

4. **Responsiveness:**
   - Fix any layout shifts on the `ClientCard` components during usage updates on mobile views.

# Hard Scope Rules
- No new features or API routes.
- Use only existing Tailwind classes and Headless UI / Radix components.

# Touches
- src/app/components/Admin/CidrDialog.vue
- src/app/components/Admin/RestartInterfaceDialog.vue
- src/app/pages/dashboard/index.vue
- src/app/pages/admin/routers/index.vue
- src/app/pages/admin/interface.vue
- src/app/components/ClientCard/index.vue
- src/app/composables/useSubmit.ts (if used for global error handling)
