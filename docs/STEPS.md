# Các bước xây dựng

Plan tổng thể ở [PLAN.md](PLAN.md). File này chia việc thành từng bước nhỏ, làm theo thứ tự.

## Quy tắc

- Mỗi lần chỉ làm một bước. Bước chỉ được đánh dấu `[x]` khi phần **Kiểm tra** đạt hết.
- Mỗi bước xong là một commit.
- Ghi kết quả kiểm tra (lệnh đã chạy, kết quả) vào dòng **Kết quả** của bước đó.
- **Em** kiểm tra: lệnh hoặc test tự động, em chạy và ghi lại. **Anh** kiểm tra: anh tự mở xem hoặc thử tay. Bước có phần của anh thì chờ anh xác nhận rồi mới sang bước sau.
- Kiểm tra không đạt thì sửa trong chính bước đó, không làm tiếp.

---

## Mốc 1: Môi trường dev

Xong mốc khi: một lệnh `pnpm dev` bật Hub, web, Postgres, Redis; `pnpm test` xanh.

- [x] **B1. Khung monorepo**
      pnpm workspace (`apps/*`, `packages/*`), `tsconfig` chung, ESLint, Prettier, `.editorconfig`.
      Kiểm tra (Em): `pnpm install` không lỗi; `pnpm lint` và `pnpm format:check` pass.
      Kết quả: 27/11/2026. Node 26.13.1, pnpm 12.23.0, TypeScript 7.0.2 (`tsc`, gói `@typescript/native`) chạy song song TypeScript 6.0 (`tsc6`, alias `typescript` → `@typescript/typescript6`) theo hướng dẫn của TS 7.0, vì TS 7.0 chưa có API mà typescript-eslint 8.74 cần, ESLint 10.15, Prettier 3.10.1. Xóa `node_modules` rồi `pnpm install --frozen-lockfile`: không lỗi, lockfile qua supply-chain policy. `pnpm lint`: exit 0. `pnpm format:check`: "All matched files use Prettier code style!". `pnpm audit`: không có lỗ hổng.

- [ ] **B2. `packages/shared` và Vitest**
      Package dùng chung có zod, một schema mẫu kèm test.
      Kiểm tra (Em): `pnpm test` chạy và xanh.
      Kết quả:

- [ ] **B3. Postgres và Redis cho dev**
      `docker-compose.dev.yml` có `postgres`, `redis`, healthcheck, volume riêng.
      Kiểm tra (Em): `docker compose -f docker-compose.dev.yml up -d`, cả hai container `healthy`.
      Kết quả:

- [ ] **B4. Hub API tối thiểu**
      `apps/hub`: Fastify, đọc env bằng zod, route `GET /api/health` kiểm tra kết nối DB và Redis.
      Kiểm tra (Em): `curl /api/health` trả `{ db: "ok", redis: "ok" }`; integration test cho route này xanh; tắt Redis thì health báo lỗi đúng.
      Kết quả:

- [ ] **B5. Drizzle và migration đầu tiên**
      Bảng `users`, `user_sessions`, `audit_log`. Lệnh `pnpm db:migrate`.
      Kiểm tra (Em): chạy migrate tạo đủ bảng; chạy lần hai không lỗi; `audit_log` bị thu hồi quyền UPDATE, DELETE.
      Kết quả:

- [ ] **B6. Web Nuxt 4**
      `apps/web` dạng SPA, proxy `/api` sang Hub, trang chủ hiện trạng thái `/api/health`.
      Kiểm tra (Em): `pnpm build` web không lỗi. (Anh): mở trình duyệt thấy trang báo DB và Redis "ok".
      Kết quả:

- [ ] **B7. Một lệnh chạy tất cả, cập nhật CLAUDE.md**
      `pnpm dev` bật DB, Redis, Hub, web. `CLAUDE.md` ghi lệnh build, lint, test, chạy một test lẻ.
      Kiểm tra (Anh): clone sạch, làm theo `CLAUDE.md`, chạy được.
      Kết quả:

---

## Mốc 2: Đăng nhập

Xong mốc khi: anh đăng nhập thật bằng mật khẩu và 2FA; mọi luồng sai đều bị chặn và có test.

- [ ] **B8. Tạo admin đầu tiên**
      Lệnh `pnpm hub user:create-admin`. Mật khẩu băm argon2id.
      Kiểm tra (Em): test băm và kiểm mật khẩu; chạy lệnh tạo được user `is_admin`.
      Kết quả:

- [ ] **B9. Đăng nhập, phiên, đăng xuất**
      `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/me`. Cookie httpOnly, Secure, SameSite, phiên lưu Postgres.
      Kiểm tra (Em): test đúng mật khẩu, sai mật khẩu, phiên hết hạn, đăng xuất xong cookie vô hiệu.
      Kết quả:

- [ ] **B10. Khóa tạm và giới hạn tốc độ**
      Sai 10 lần thì khóa tạm; rate limit theo IP.
      Kiểm tra (Em): test lần sai thứ 10 bị khóa, hết hạn khóa đăng nhập lại được.
      Kết quả:

- [ ] **B11. 2FA TOTP**
      Lần đầu đăng nhập bắt buộc thiết lập (QR); các lần sau phải nhập mã. Secret TOTP mã hóa.
      Kiểm tra (Em): test mã đúng, mã sai, mã dùng lại. (Anh): quét QR bằng app thật.
      Kết quả:

- [ ] **B12. Ghi audit cho đăng nhập**
      Đăng nhập thành công, thất bại, khóa, đăng xuất đều vào `audit_log`.
      Kiểm tra (Em): test từng sự kiện tạo đúng một dòng audit.
      Kết quả:

- [ ] **B13. Màn 1, 2, 15**
      Đăng nhập, xác thực 2FA, Tài khoản của tôi (đổi mật khẩu, đặt lại 2FA, đăng xuất).
      Kiểm tra (Em): E2E Playwright cho luồng đăng nhập đủ 2 bước. (Anh): đăng nhập thật trên trình duyệt.
      Kết quả:

---

## Mốc 3: Người dùng và dự án

Cần chốt trước B15: dự án có 2 vai trò (Quản lý, Thành viên) hay 3 (thêm Xem).

Xong mốc khi: admin mời được người, tạo dự án, gán thành viên; người không được gán không thấy dự án.

- [ ] **B14. Bảng dữ liệu còn lại của Hub**
      `invites`, `projects`, `project_members`, `servers`, `settings`.
      Kiểm tra (Em): migrate chạy được, chạy lại không lỗi.
      Kết quả:

- [ ] **B15. Policy quyền**
      `apps/hub/src/auth/policy.ts`, mọi route đi qua đây.
      Kiểm tra (Em): test ma trận vai trò × hành động theo bảng ở PLAN.md mục 5.
      Kết quả:

- [ ] **B16. Người dùng và link mời**
      Màn 11 (admin quản lý người dùng), màn 3 (kích hoạt qua link mời 24 giờ).
      Kiểm tra (Em): test link hết hạn, link dùng lại, khóa và mở khóa tài khoản. (Anh): mời một người thật, người đó kích hoạt được.
      Kết quả:

- [ ] **B17. Dự án và thành viên**
      Màn 4 (danh sách dự án), màn 9 (thành viên), màn 10 bước 1–2 (tạo dự án: thông tin, thành viên).
      Kiểm tra (Em): người không được gán gọi API dự án bị 403. (Anh): tạo dự án, gán người, đăng nhập bằng người đó kiểm tra chỉ thấy dự án được gán.
      Kết quả:

- [ ] **B18. Cài đặt hệ thống và nhật ký**
      Màn 12 phần cơ bản, màn 13 (xem và lọc audit).
      Kiểm tra (Em): chỉ admin vào được. (Anh): lọc audit theo người và hành động.
      Kết quả:

---

## Mốc 4: Cài server nhỏ qua SSH

Xong mốc khi: tạo dự án trỏ vào một server, Hub cài xong, daemon báo xanh; chạy được trên server thật của anh.

- [ ] **B19. Server giả lập**
      Container `sandbox-server` (Ubuntu, sshd, Docker bên trong) trong `docker-compose.dev.yml`.
      Kiểm tra (Em): SSH root bằng key vào được; `docker run hello-world` chạy được bên trong.
      Kết quả:

- [ ] **B20. Mã hóa secret**
      AES-256-GCM, master key từ env, bảng `secrets`.
      Kiểm tra (Em): test mã hóa rồi giải mã đúng; sai key hoặc dữ liệu bị sửa thì báo lỗi.
      Kết quả:

- [ ] **B21. Kết nối SSH và preflight**
      Nhận hoặc sinh key ed25519, ghim host key, chạy `preflight.sh` (OS, RAM, ổ đĩa, cổng).
      Kiểm tra (Em): với `sandbox-server`: preflight ra báo cáo; host key bị đổi thì Hub từ chối kết nối.
      Kết quả:

- [ ] **B22. `install.sh` và log trực tiếp**
      Cài Docker, git, user `agent`, Node.js, Claude Code, cứng hóa sshd, thêm `from=` vào `authorized_keys`. Log stream lên màn 10 bước 3–4.
      Kiểm tra (Em): chạy hai lần trên `sandbox-server` không lỗi, không cài trùng; mỗi phiên SSH có trong audit.
      Kết quả:

- [ ] **B23. Daemon rỗng**
      Node Agent chạy bằng systemd, chỉ gửi heartbeat; màn 4 hiện trạng thái xanh hoặc đỏ.
      Kiểm tra (Em): dừng daemon thì màn 4 đổi đỏ trong vòng 30 giây.
      Kết quả:

- [ ] **B24. Thử trên server thật**
      Kiểm tra (Anh): tạo server nhỏ, gắn key, tạo dự án, cài xong, daemon xanh.
      Kết quả:

---

## Các mốc sau

Chia bước chi tiết khi làm tới, theo PLAN.md mục 7:

- Mốc 5: Node Agent và Web Chat (đăng nhập Claude bằng `setup-token`, chạy task, 3 cổng duyệt).
- Mốc 6: Cloudflare tự động (tunnel, DNS, Access).
- Mốc 7: vận hành nhiều dự án.
- Mốc riêng: "não" cho Wepro (GĐ0), làm khi repo Wepro có trên máy.
