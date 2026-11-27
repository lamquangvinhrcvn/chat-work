# Kế hoạch phát triển chat-work (Agent Dev Platform)

Tài liệu này biến bản kiến trúc (artifact "Agent Dev Platform", bản 3.1) thành kế hoạch làm việc: công nghệ, container, màn hình, dữ liệu, các giai đoạn và tiêu chí đạt.

Cập nhật: 27/11/2026

---

## 1. Tóm tắt

- **Hub** là ứng dụng web chạy ở `agent-wepro.vdtvn.com`. Người dùng đăng nhập, chat với agent của từng dự án, xem agent làm việc theo thời gian thực, bấm duyệt.
- Hub có danh sách người dùng riêng, không dùng tài khoản hay phòng ban của Wepro. Admin tạo người dùng và gán quyền theo dự án.
- **Mỗi dự án = một server nhỏ** do anh tự tạo (IP public, mở cổng 22). Admin nhập IP và key SSH root vào Hub, Hub tự cài Docker, Node Agent, Claude Code, cloudflared.
- **Node Agent** trên server nhỏ chạy Claude Code CLI (headless) bằng tài khoản Claude đăng nhập sẵn cho dự án đó. Agent code, chạy test thật, mở Merge Request trên GitLab.
- **Web dev** của dự án mở qua Cloudflare Tunnel: `aw-<slug>-<tên-port>.vdtvn.com`, có Cloudflare Access chặn người ngoài.
- **Xóa dự án**: Hub dừng task, gỡ tunnel, DNS, Access, gỡ daemon và key của mình. Folder và container trên server giữ nguyên.
- **Dự án đầu tiên**: một tính năng của Wepro.

---

## 2. Công nghệ

Một ngôn ngữ cho cả hệ thống: **TypeScript**. Phiên bản cụ thể chốt lúc khởi tạo repo (lấy bản ổn định mới nhất, kiểm tra release và advisory), rồi ghim trong `package.json` và `pnpm-lock.yaml`.

### 2.1 Hub

| Tầng           | Công nghệ                                                                                                       | Ghi chú                                              |
| -------------- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Web UI         | Nuxt 4 chế độ SPA (`ssr: false`), Nuxt UI, Pinia, `socket.io-client`, `markdown-it` + Shiki, Monaco diff editor | Build ra file tĩnh, Hub phục vụ luôn. Không cần SEO. |
| API + realtime | Node.js (Active LTS), Fastify, Socket.IO + `@socket.io/redis-adapter`                                           | Một tiến trình cho REST, socket và worker.           |
| Hàng đợi       | BullMQ (+ Bull Board cho admin)                                                                                 | Mỗi dự án một hàng đợi, giới hạn task song song.     |
| ORM, schema    | Drizzle ORM + drizzle-kit (migration), zod                                                                      | Schema zod dùng chung trong `packages/shared`.       |
| Đăng nhập      | `@node-rs/argon2`, `otplib` (TOTP), cookie phiên lưu Postgres                                                   | Tự viết, không OIDC.                                 |
| SSH            | `ssh2`                                                                                                          | Provisioner chạy bộ script cố định.                  |
| Cloudflare     | `cloudflare` (SDK chính thức)                                                                                   | Tunnel, DNS, Access.                                 |
| GitLab         | REST API qua `fetch`                                                                                            | Webhook MR merged, mở MR nếu Node không tự mở.       |
| Log            | pino                                                                                                            | JSON log, xem bằng `docker logs`.                    |
| Mã hóa secret  | `node:crypto` AES-256-GCM                                                                                       | Master key trong env, không nằm trong DB.            |

### 2.2 Node Agent (trên server nhỏ)

| Thành phần      | Công nghệ                                                                                                           | Ghi chú                                                                                                                    |
| --------------- | ------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Daemon          | Node.js (Active LTS), `socket.io-client`, `better-sqlite3` (outbox), systemd                                        | Chạy dưới user `agent`, thuộc group `docker`.                                                                              |
| Runner          | Claude Code CLI: `claude -p --output-format stream-json --input-format stream-json --verbose`, `--resume <session>` | Tài khoản Claude của dự án, token qua `claude setup-token`. Runner tách lớp để đổi sang Agent SDK + API key bằng cấu hình. |
| Quyền thao tác  | Hook `PreToolUse` gọi HTTP cục bộ tới daemon, daemon hỏi Hub                                                        | Thay cho `canUseTool` của SDK.                                                                                             |
| Git             | `git worktree`, deploy key `ed25519` sinh trên server                                                               | Branch `agent/<task>`.                                                                                                     |
| Môi trường test | Docker Compose của dự án, mỗi task một compose project                                                              | Compose nhận đường dẫn worktree qua biến môi trường.                                                                       |
| Tunnel          | `cloudflared` (systemd)                                                                                             | Token do Hub tạo qua API.                                                                                                  |

### 2.3 Công cụ phát triển

pnpm workspaces, Vitest (unit + integration), Playwright (E2E cho web), ESLint, Prettier, Docker Compose cho môi trường dev.

---

## 3. Docker container

### 3.1 Server Hub (`deploy/docker-compose.hub.yml`)

| Container               | Image                                             | Vai trò                                                         | Volume                                            | Cổng                        |
| ----------------------- | ------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------- | --------------------------- |
| `hub`                   | build từ `apps/hub` (đã gồm bản build `apps/web`) | REST, Socket.IO, worker BullMQ, provisioner SSH, gọi Cloudflare | `hub-data` (file tạm, artifact khi chưa có MinIO) | 3000, chỉ trong mạng nội bộ |
| `postgres`              | `postgres` (bản ổn định mới nhất)                 | Nguồn dữ liệu chính                                             | `pg-data`                                         | không mở ra ngoài           |
| `redis`                 | `redis` (bản ổn định mới nhất)                    | Hàng đợi BullMQ, pub/sub Socket.IO                              | `redis-data` (AOF bật)                            | không mở ra ngoài           |
| `cloudflared`           | `cloudflare/cloudflared`                          | Tunnel cho `agent-wepro.vdtvn.com` → `hub:3000`                 | ·                                                 | không mở ra ngoài           |
| `pgbackup`              | `postgres` + cron script                          | `pg_dump` mỗi đêm, đẩy lên R2, giữ 14 bản                       | ·                                                 | ·                           |
| `minio` _(giai đoạn 4)_ | `minio/minio`                                     | Ảnh Playwright, log test, báo cáo                               | `minio-data`                                      | không mở ra ngoài           |

**Tổng: 5 container lúc đầu, 6 khi thêm MinIO.** Server Hub không mở cổng nào ra internet. Mọi truy cập đi qua Cloudflare Tunnel.

### 3.2 Server nhỏ của dự án

Hub **không** chạy Node Agent trong container. Daemon cần dùng Docker của host, git và filesystem trực tiếp, nên chạy bằng systemd.

| Thành phần                     | Kiểu                  | Ghi chú                                                                                                                                                           |
| ------------------------------ | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Docker Engine + compose plugin | gói hệ thống          | Hub cài qua `get.docker.com`.                                                                                                                                     |
| `node-agent.service`           | systemd, user `agent` | Daemon.                                                                                                                                                           |
| `cloudflared.service`          | systemd, root         | Tunnel riêng của server.                                                                                                                                          |
| Compose của dự án              | container             | Với Wepro: app, web server, DB, Redis, Mailpit… theo `docker-compose.yml` của Wepro. Mỗi task thêm một compose project riêng (`<slug>-<task>`) để test song song. |

### 3.3 Môi trường dev của chính repo này (`docker-compose.dev.yml`)

| Container           | Vai trò                                                                                                                                                         |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `postgres`, `redis` | Như production.                                                                                                                                                 |
| `sandbox-server`    | Ubuntu có `sshd` và Docker-in-Docker (privileged). Giả lập một server nhỏ để test provisioner và Node Agent mà không cần VM thật. Chỉ dùng khi dev và trong CI. |

`hub` và `web` chạy trực tiếp bằng `pnpm dev` trên máy dev để có hot reload.

---

## 4. Màn hình

**Tổng: 15 màn hình**, cộng các modal và drawer liệt kê ở cột cuối. Cột GĐ là giai đoạn làm (mục 7).

| #   | Màn hình            | Route                 | Ai thấy                    | Nội dung chính                                                                                                                                                                  | Modal / drawer                                                        | GĐ  |
| --- | ------------------- | --------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | --- |
| 1   | Đăng nhập           | `/login`              | Mọi người                  | Email, mật khẩu, khóa tạm sau 10 lần sai                                                                                                                                        | ·                                                                     | 1   |
| 2   | Xác thực 2FA        | `/login/2fa`          | Mọi người                  | Nhập mã TOTP                                                                                                                                                                    | ·                                                                     | 1   |
| 3   | Kích hoạt tài khoản | `/invite/:token`      | Người được mời             | Đặt mật khẩu, quét QR thiết lập TOTP, mã khôi phục                                                                                                                              | ·                                                                     | 1   |
| 4   | Danh sách dự án     | `/`                   | Người đã đăng nhập         | Thẻ dự án mình được gán: trạng thái server, task đang chạy, task chờ duyệt                                                                                                      | ·                                                                     | 1   |
| 5   | Chat dự án          | `/p/:slug`            | Thành viên dự án           | Cột trái: danh sách thread (lọc theo trạng thái). Giữa: thread đang mở                                                                                                          | Tạo task mới (loại Sửa code / Hỏi đáp)                                | 2   |
| 6   | Thread task         | `/p/:slug/t/:taskId`  | Thành viên dự án           | Stream realtime, tool call thu gọn, kết quả test, thẻ cổng duyệt, ô nhắn thêm, nút Dừng, thanh trạng thái task                                                                  | Drawer diff (Monaco), drawer plan, modal yêu cầu quyền thao tác       | 2   |
| 7   | Tổng quan dự án     | `/p/:slug/overview`   | Thành viên dự án           | Server online/offline, container đang chạy, danh sách URL web dev, tài khoản Claude (đã đăng nhập, hạn token, lần chạm hạn mức gần nhất)                                        | ·                                                                     | 2   |
| 8   | Cấu hình dự án      | `/p/:slug/settings`   | Admin, Quản lý             | Tab Repo (deploy key, GitLab token), tab Môi trường (file compose, port), tab Claude (đăng nhập, đăng nhập lại), tab Server (chẩn đoán, cài lại, đổi key SSH, khóa Node)        | Modal Đăng nhập Claude (link + ô dán mã), modal xác nhận host key mới | 2–3 |
| 9   | Thành viên dự án    | `/p/:slug/members`    | Admin (sửa), Quản lý (xem) | Danh sách thành viên, vai trò                                                                                                                                                   | Modal thêm thành viên                                                 | 1   |
| 10  | Tạo dự án (wizard)  | `/admin/projects/new` | Admin                      | Bước 1 thông tin + repo · bước 2 IP + key SSH · bước 3 xác nhận host key · bước 4 tiến trình cài đặt (log realtime) · bước 5 deploy key · bước 6 port · bước 7 đăng nhập Claude | ·                                                                     | 1–3 |
| 11  | Quản lý người dùng  | `/admin/users`        | Admin                      | Danh sách, tạo, khóa, reset 2FA, thu hồi phiên                                                                                                                                  | Modal tạo người dùng (hiện link mời)                                  | 1   |
| 12  | Cài đặt hệ thống    | `/admin/settings`     | Admin                      | API token Cloudflare (kiểm tra quyền), webhook Slack, ngân sách ngày, IP Hub                                                                                                    | ·                                                                     | 1–3 |
| 13  | Nhật ký audit       | `/admin/audit`        | Admin                      | Lọc theo người, dự án, hành động. Xem output phiên SSH                                                                                                                          | Drawer chi tiết                                                       | 2   |
| 14  | Chi phí & hoạt động | `/admin/usage`        | Admin                      | Task, MR, thời gian, lần chạm hạn mức theo dự án và theo ngày                                                                                                                   | ·                                                                     | 4   |
| 15  | Tài khoản của tôi   | `/me`                 | Người đã đăng nhập         | Đổi mật khẩu, đặt lại 2FA, danh sách phiên đăng nhập                                                                                                                            | ·                                                                     | 1   |

Xóa dự án là một nút trong màn 8 (tab Server), mở modal xác nhận gõ lại slug và tùy chọn dừng container.

---

## 5. Người dùng và phân quyền

- Vai trò toàn hệ thống: **Admin**.
- Vai trò theo dự án: **Quản lý**, **Thành viên**, có thể thêm **Xem**. _Chưa chốt dùng 2 hay 3 vai trò._ Cột `role` là enum nên đổi sau không ảnh hưởng cấu trúc.

| Hành động                                                     | Admin | Quản lý | Thành viên    | Xem |
| ------------------------------------------------------------- | ----- | ------- | ------------- | --- |
| Tạo người dùng, tạo và xóa dự án, gán thành viên              | ✓     |         |               |     |
| Cấu hình dự án: port, compose, đăng nhập Claude, token GitLab | ✓     | ✓       |               |     |
| Tạo task, chat, hỏi đáp                                       | ✓     | ✓       | ✓             |     |
| Duyệt plan (cổng 1)                                           | ✓     | ✓       | ✓             |     |
| Duyệt merge, production (cổng 2, 3)                           | ✓     | ✓       |               |     |
| Dừng task                                                     | ✓     | ✓       | task mình tạo |     |
| Xem lịch sử, mở web dev                                       | ✓     | ✓       | ✓             | ✓   |

Kiểm tra quyền ở **một chỗ** (`apps/hub/src/auth/policy.ts`), dùng cho cả REST và mọi event Socket.IO.

---

## 6. Cấu trúc repo và dữ liệu

### 6.1 Repo

```
chat-work/
├─ apps/
│  ├─ web/          Nuxt 4 SPA (15 màn hình ở mục 4)
│  ├─ hub/          Fastify + Socket.IO + BullMQ + Drizzle
│  │  └─ src/
│  │     ├─ auth/        đăng nhập, 2FA, phiên, policy quyền
│  │     ├─ projects/    dự án, thành viên, port
│  │     ├─ provision/   ssh2 client, chạy script, ghim host key
│  │     ├─ cloudflare/  tunnel, ingress, DNS, Access
│  │     ├─ tasks/       máy trạng thái task, hàng đợi, cổng duyệt
│  │     ├─ socket/      namespace /ui và /node
│  │     ├─ gitlab/      webhook, API
│  │     └─ db/          schema Drizzle, migration
│  └─ node-agent/   daemon, runner Claude Code CLI, outbox, worktree, agent-env
├─ packages/
│  └─ shared/       schema zod: event socket, trạng thái task, payload, hằng số
├─ deploy/
│  ├─ docker-compose.hub.yml
│  └─ scripts/      preflight.sh, install.sh, update-daemon.sh, diagnose.sh, detach.sh
├─ docker-compose.dev.yml
├─ docs/PLAN.md
└─ CLAUDE.md
```

### 6.2 Bảng Postgres

| Bảng                               | Cột chính                                                                                                         |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `users`                            | email, tên, `password_hash`, `totp_secret_enc`, `is_admin`, `status`, `failed_logins`, `locked_until`             |
| `user_sessions`                    | `user_id`, `token_hash`, IP, user agent, `expires_at`, `revoked_at`                                               |
| `invites`                          | `user_id`, `token_hash`, `expires_at`, `used_at`                                                                  |
| `projects`                         | `slug`, tên, `repo_url`, `default_branch`, `compose_file`, `max_parallel`, `status` (active / archived)           |
| `project_members`                  | `project_id`, `user_id`, `role`                                                                                   |
| `servers`                          | `project_id`, host, `ssh_port`, `host_key_fp`, `ssh_key_enc`, `node_token_hash`, `last_seen_at`, `daemon_version` |
| `project_ports`                    | `project_id`, tên, port, hostname, `apply_status`                                                                 |
| `cf_resources`                     | `project_id`, `kind` (tunnel / dns / access_app), `cf_id`                                                         |
| `secrets`                          | `project_id`, `kind` (gitlab_token), `value_enc`                                                                  |
| `claude_auth`                      | `project_id`, trạng thái, email tài khoản, `token_expires_at`, `last_limit_at`, `limit_resets_at`                 |
| `tasks`                            | `project_id`, `kind` (change / ask), tiêu đề, `state`, `branch`, `mr_url`, `created_by`                           |
| `messages`                         | `task_id`, `seq`, `role`, nội dung, `tsv` (full-text)                                                             |
| `events`                           | `task_id`, `seq`, `ts`, `kind`, `payload` jsonb. Khóa chính `(task_id, seq)`, partition theo tháng                |
| `phases`                           | `task_id`, tên pha, trạng thái, thời gian, `session_id`, số turn                                                  |
| `approvals`, `permission_requests` | cổng, quyết định, ghi chú, người duyệt · tool, input, kết quả                                                     |
| `artifacts`                        | `task_id`, loại, `storage_key`, kích thước                                                                        |
| `settings`                         | khóa, giá trị (giá trị nhạy cảm mã hóa)                                                                           |
| `audit_log`                        | thời gian, người hoặc Node, hành động, đối tượng, dữ liệu. Chỉ thêm, thu hồi UPDATE và DELETE                     |

### 6.3 Biến môi trường của Hub

`DATABASE_URL`, `REDIS_URL`, `MASTER_KEY` (32 byte, mã hóa secret), `SESSION_SECRET`, `PUBLIC_URL=https://agent-wepro.vdtvn.com`, `HUB_EGRESS_IP` (dùng cho `from=` và lọc IP token Cloudflare), `CLOUDFLARE_TUNNEL_TOKEN` (tunnel của Hub), `SLACK_WEBHOOK_URL`, `R2_*` (backup). API token Cloudflare nhập qua màn 12, lưu mã hóa trong `settings`.

---

## 7. Các giai đoạn

Mỗi giai đoạn chỉ xong khi đạt hết tiêu chí. Thời gian là ước lượng cho 1 người cùng agent.

### Giai đoạn 0: "Não" cho Wepro (1 tuần)

Làm trên repo Wepro, chưa cần Hub.

- [ ] `CLAUDE.md` cho Wepro: kiến trúc, quy ước, lệnh test, điều cấm.
- [ ] Skill `feature-workflow` (spec → plan → TDD → verify), subagents `planner`, `implementer`, `test-writer`, `reviewer`, `security-reviewer`.
- [ ] Hooks: format sau Edit, test liên quan khi Stop, chặn lệnh nguy hiểm. Permissions trong `.claude/settings.json`.
- [ ] Compose của Wepro nhận đường dẫn worktree và tên project qua biến môi trường. Script `agent-env up|down <task>` dựng và dọn DB test từ bản dump đã che dữ liệu.

**Đạt:** chạy `claude` trong worktree, 3 task nhỏ tự ra MR có test pass. Hai worktree chạy test cùng lúc không giẫm nhau.

### Giai đoạn 1: Hub nền và cài server qua SSH (1–2 tuần)

- [ ] Monorepo pnpm, `packages/shared`, ESLint, Prettier, Vitest, `docker-compose.dev.yml` có `sandbox-server`.
- [ ] Schema và migration: `users`, `user_sessions`, `invites`, `projects`, `project_members`, `servers`, `settings`, `audit_log`.
- [ ] Đăng nhập, 2FA, link mời, phiên, khóa tạm, policy quyền.
- [ ] Màn 1, 2, 3, 4, 9, 11, 15. Màn 10 bước 1–4. Màn 12 phần cơ bản.
- [ ] Provisioner: sinh key hoặc nhận private key, mã hóa, ghim host key, chạy `preflight.sh`, `install.sh` (idempotent), stream log lên UI, ghi audit.
- [ ] `install.sh` cài Docker, git, user `agent`, Node.js, Claude Code, daemon rỗng (chỉ heartbeat), cứng hóa `sshd`, thêm `from=` vào `authorized_keys`.

**Đạt:**

- Tạo dự án trỏ vào `sandbox-server`, cài xong, daemon báo xanh trên màn 4.
- Chạy `install.sh` lần hai không lỗi, không cài trùng.
- Người không được gán không thấy dự án, gọi API trực tiếp bị 403.
- Test: unit cho policy quyền và mã hóa; integration cho luồng đăng nhập và provisioner với `sandbox-server`.

### Giai đoạn 2: Node Agent và Web Chat (2–3 tuần)

- [ ] Socket.IO `/node`: xác thực token Node, heartbeat, `project:sync`, `claude:login`, `task:start`, `task:input`, `task:abort`, `agent:events`, `phase:done`, `env:status`.
- [ ] Outbox SQLite, `seq` theo task, ack, gửi bù khi kết nối lại.
- [ ] Đăng nhập Claude: daemon chạy `claude setup-token`, gửi link lên UI, nhận mã dán lại, lưu token trên server. Kiểm tra sớm trong tuần đầu; không chạy được thì làm phương án terminal web tạm thời.
- [ ] Runner Claude Code CLI theo pha: plan (chỉ đọc) → code → test → review (session mới) → MR. Hook `PreToolUse` hỏi quyền qua Hub.
- [ ] Máy trạng thái task, 3 cổng duyệt, `needs_input`, `paused_limit` (tự chạy tiếp khi hạn mức làm mới), kill switch.
- [ ] Deploy key sinh trên server (màn 10 bước 5), clone repo, webhook GitLab cho MR merged.
- [ ] Socket.IO `/ui`, `task:subscribe` với `afterSeq`.
- [ ] Màn 5, 6, 7, 8 (tab Repo, Claude, Server), 13.

**Đạt:**

- Một task Wepro chạy trọn từ màn 6 đến MR trên GitLab.
- Rút dây mạng giữa Node và Hub khi task đang chạy: không mất event, UI hiện đủ khi kết nối lại.
- Đóng trình duyệt rồi mở lại thread: thấy đủ lịch sử.
- Bấm Dừng: process Claude và compose của task dừng trong 5 giây.

### Giai đoạn 3: Cloudflare tự động (3–5 ngày)

- [ ] Màn 12: nhập API token, Hub kiểm tra đủ 3 quyền.
- [ ] Tạo tunnel khi cài server, cài `cloudflared` qua SSH.
- [ ] Màn 8 tab Môi trường và màn 10 bước 6: khai báo port, Hub ghi ingress, CNAME, Access app theo email thành viên.
- [ ] Đồng bộ Access policy khi thêm hoặc bớt thành viên.
- [ ] Luồng xóa dự án đầy đủ (dọn Cloudflare, `detach.sh`, gỡ key, lưu trữ dữ liệu).

**Đạt:**

- Lưu port thì URL `aw-<slug>-<tên>.vdtvn.com` chạy trong 1 phút, hot reload của Vite chạy qua tunnel.
- Người không phải thành viên bị Access chặn.
- Xóa dự án: không còn tunnel, DNS, Access app thừa trên Cloudflare; folder và container trên server còn nguyên.

### Giai đoạn 4: Vận hành nhiều dự án (1 tuần)

- [ ] Dự án thứ hai.
- [ ] MinIO cho ảnh và log test, màn 14.
- [ ] Báo cáo Slack hằng ngày, cảnh báo đĩa server trên 80%, token Claude sắp hết hạn.
- [ ] `pgbackup` và thử khôi phục.
- [ ] Nút đổi key SSH, `update-daemon.sh`, `diagnose.sh`.

**Đạt:** hai dự án chạy task cùng lúc ổn định một tuần, khôi phục được bản backup vào DB tạm.

**Tổng: khoảng 5–7 tuần tới bản dùng được (giai đoạn 0–3).**

---

## 8. Kiểm thử

| Loại        | Công cụ                           | Phạm vi                                                                                |
| ----------- | --------------------------------- | -------------------------------------------------------------------------------------- |
| Unit        | Vitest                            | policy quyền, mã hóa, máy trạng thái task, tạo ingress Cloudflare, lọc secret          |
| Integration | Vitest + `docker-compose.dev.yml` | đăng nhập, provisioner với `sandbox-server`, outbox và ack, Cloudflare API (mock HTTP) |
| E2E         | Playwright                        | đăng nhập + 2FA, tạo dự án, tạo task tới cổng 1, quyền theo vai trò                    |
| Thử thật    | server nhỏ + Wepro                | tiêu chí "Đạt" của từng giai đoạn                                                      |

---

## 9. Câu hỏi còn mở

1. **Vai trò trong dự án: 2 hay 3?** Quản lý + Thành viên, hoặc thêm Xem.
2. **Mọi dự án đều ở GitLab công ty?** Có GitHub thì thêm client thứ hai.
3. **Transcript đầy đủ chỉ để trên server nhỏ, hay cần bản sao tập trung?** Đề xuất hiện tại: chỉ để trên server.
4. **Tài khoản Claude cho từng dự án:** seat Team hay gói Max riêng? Cần kiểm tra điều khoản hiện hành của Anthropic về dùng tài khoản gói cho việc chạy tự động trước khi chạy thật.
