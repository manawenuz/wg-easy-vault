You are an implementation engineer working on the manawenuz/wg-easy fork.
Your task is to implement SSH passphrase support and align AmneziaWG handling with the `w0rng/amnezia-wg-easy` approach (Docker/Userspace focus).

# Tasks

1. **Database & Transport (SSH Passphrase):** 
   - Update `src/server/database/schema.ts` to add `sshPassphraseEncrypted` (text) to the `router` table.
   - Create migration `src/server/database/migrations/0007_router_passphrase.sql`.
   - Update `src/server/transports/ssh.ts` to handle the passphrase in `SshAuth`.
   - Update Router POST/PATCH/Test APIs to handle the encrypted passphrase (match `credentialsEncrypted` pattern).

2. **AmneziaWG Userspace Fallback:**
   - Update the `Dockerfile` to set `ENV WG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go` and `ENV WG_I_PREFER_USERSPACE_TO_KERNEL=1`. This ensures it works even if the host kernel lacks the AWG module (matching w0rng behavior).
   - Update `src/server/engines/amneziawg/index.ts` to log a clear warning if the kernel module is missing but we are falling back to the userspace implementation.

3. **Engine Transparency & Docker Detection:**
   - Update `src/server/api/admin/engines.get.ts`.
   - If `awg` binary is missing on the PATH, check if `docker` is available.
   - If `docker` is available, return the engine status as "Available (via Docker)" and include an error message explaining that host binaries are missing.

4. **Remote Linux "Docker Engine" Fallback:**
   - Update `AmneziaWgEngine` to handle cases where `awg` is missing on a remote Linux host (via SSH) but `docker` is present.
   - In this case, wrap the `awg-quick` and `awg` commands in a `docker run --rm --cap-add=NET_ADMIN --network=host -v /etc/amnezia:/etc/amnezia -v /lib/modules:/lib/modules ghcr.io/amnezia-vpn/amneziawg-tools` execution. This allows AmneziaWG to work on Debian without host-side tools.

5. **User Interface:**
   - Update `src/app/pages/admin/routers/[id]/index.vue` to add a "SSH Key Passphrase" input field (password type) in the SSH credentials section.

# Hard Scope Rules
- Match existing encryption/decryption patterns.
- Do not break existing local-shell WireGuard functionality.

# Touches
- src/server/database/schema.ts
- src/server/database/migrations/0007_router_passphrase.sql (new)
- src/server/transports/ssh.ts
- src/server/api/admin/router/index.post.ts
- src/server/api/admin/router/[id]/index.patch.ts
- src/server/api/admin/router/[id]/test.post.ts
- src/server/api/admin/engines.get.ts
- src/server/engines/amneziawg/index.ts
- Dockerfile
- src/app/pages/admin/routers/[id]/index.vue
