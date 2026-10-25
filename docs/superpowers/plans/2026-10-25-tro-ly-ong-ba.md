# Trợ Lý Ông Bà Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** App Android cho người lớn tuổi: gọi/nhắn/Zalo người thân bằng nút to, sổ tài khoản mã hoá trên máy, hướng dẫn từng bước có đọc to.

**Architecture:** Expo + expo-router, mỗi tính năng là một thư mục trong `src/features/` gồm phần logic thuần (có test) và màn hình trong `app/`. Dữ liệu nhạy cảm nằm trong `expo-secure-store`, phần còn lại trong AsyncStorage. Supabase chỉ cung cấp nội dung hướng dẫn (đọc công khai), có cache và dữ liệu mẫu đóng gói sẵn để chạy offline.

**Tech Stack:** Expo SDK (latest stable), TypeScript strict, expo-router, expo-secure-store, expo-local-authentication, expo-contacts, expo-intent-launcher, expo-linking, expo-speech, expo-clipboard, @react-native-async-storage/async-storage, @supabase/supabase-js, jest-expo, @testing-library/react-native, EAS Build.

**Spec:** `docs/superpowers/specs/2026-10-25-tro-ly-ong-ba-design.md`

## Global Constraints

- Android package id: `vn.trolyongba.app`; app name hiển thị: `Trợ Lý Ông Bà`.
- Mọi chữ hiển thị là tiếng Việt; câu báo lỗi luôn kèm một nút hành động.
- Cỡ chữ gốc 22, nút cao tối thiểu 72 — cả hai nhân với `fontScale` (1 | 1.25 | 1.5).
- Mật khẩu/tên đăng nhập trong Sổ tài khoản chỉ nằm trong SecureStore; không `console.log`, không gửi mạng.
- Không khai báo quyền `SEND_SMS`, `CALL_PHONE`, `READ_CALL_LOG`; chặn chúng trong `android.blockedPermissions`.
- Package ngoài: Zalo `com.zing.zalo`, VNeID `com.vnid` (xác minh lại trên máy thật ở Task 8).
- Khoá SecureStore chỉ dùng ký tự `[A-Za-z0-9._-]`.
- Supabase: chỉ dùng publishable key (`sb_publishable_…`, không bao giờ dùng secret/service_role key trong app), đọc qua biến môi trường `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY`. Project đã có: URL `https://pxfhjqrhuscuknfiasrg.supabase.co`; giá trị key do người dùng cung cấp, chỉ ghi vào `.env` (đã nằm trong `.gitignore`), `.env.example` để trống.

## Review Focus

1. Số điện thoại có `+84`, dấu cách, gạch ngang, dấu chấm, hoặc số bàn → link `tel:`/`sms:`/Zalo vẫn đúng (Task 2 test `normalizes messy numbers`).
2. Máy chưa cài Zalo/VNeID → mở trang CH Play, không crash (Task 2 test `openApp falls back to store`).
3. Lần đầu mở app khi không có mạng → vẫn thấy hướng dẫn mẫu (Task 6 test `returns seed when offline and no cache`).
4. Ông bà từ chối quyền danh bạ → câu giải thích + nút "Mở cài đặt" (Task 3 test `shows settings button when permission denied`).
5. Máy không cài khoá màn hình/vân tay → Sổ tài khoản mở bằng PIN của app thay vì bị khoá vĩnh viễn (Task 4 test `falls back to app PIN when device has no security`).

---

### Task 1: Scaffold, theme, nút to

**Files:**
- Create: project qua `npx create-expo-app@latest . --template blank-typescript`
- Create: `app/_layout.tsx`, `app/index.tsx` (tạm: một BigButton)
- Create: `src/ui/theme.ts`, `src/ui/BigButton.tsx`, `src/ui/Screen.tsx`, `src/ui/AppText.tsx`
- Create: `src/features/settings/settingsStore.ts`, `src/features/settings/SettingsContext.tsx`
- Create: `jest.config.js`, `jest.setup.ts` (mock AsyncStorage chính thức + mock SecureStore bằng `Map` trong bộ nhớ, dùng chung cho các task sau)
- Test: `src/ui/__tests__/BigButton.test.tsx`, `src/features/settings/__tests__/settingsStore.test.ts`

**Interfaces:**
- Produces:
  - `theme`: `{ colors: { bg, text, primary, call: '#1B8A3A', sms: '#1565C0', zalo: '#0068FF', danger: '#C62828', muted }, baseFont: 22, minTouch: 72, radius: 20 }`
  - `type FontScale = 1 | 1.25 | 1.5`
  - `loadSettings(): Promise<{ fontScale: FontScale }>` (mặc định `1.25`), `saveSettings(s): Promise<void>` — khoá AsyncStorage `settings.v1`
  - `useSettings(): { fontScale: FontScale; setFontScale(s: FontScale): void }`
  - `<BigButton label: string; onPress(): void; color?: string; icon?: ReactNode; accessibilityHint?: string />`
  - `<AppText size?: 'normal' | 'large' | 'title'>` — nhân `baseFont` với `fontScale` (normal ×1, large ×1.3, title ×1.6)
  - `<Screen title: string>` — SafeArea, nền `theme.colors.bg`, tiêu đề to, nút "◀ Quay lại" to khi không phải màn hình gốc

- [ ] **Step 1: Scaffold + git init + cài dependency** (`npx expo install` cho mọi lib ở Tech Stack; dev: `jest-expo jest @testing-library/react-native @types/jest`). Bật `"strict": true`. Thêm script `"test": "jest"`, `"typecheck": "tsc --noEmit"`.
- [ ] **Step 2: Write the failing tests**

```tsx
test('BigButton renders label and fires onPress', () => {
  const onPress = jest.fn();
  render(<BigButton label="Gọi điện" onPress={onPress} />);
  fireEvent.press(screen.getByText('Gọi điện'));
  expect(onPress).toHaveBeenCalledTimes(1);
});
test('BigButton min height scales with fontScale', () => {
  renderWithSettings(<BigButton label="A" onPress={() => {}} />, { fontScale: 1.5 });
  expect(screen.getByRole('button')).toHaveStyle({ minHeight: 108 });
});
test('loadSettings defaults to 1.25 and round-trips', async () => {
  expect(await loadSettings()).toEqual({ fontScale: 1.25 });
  await saveSettings({ fontScale: 1.5 });
  expect(await loadSettings()).toEqual({ fontScale: 1.5 });
});
```

- [ ] **Step 3: Run** `npm test` → FAIL (module not found)
- [ ] **Step 4: Implement** các file trên. `BigButton` dùng `Pressable` với `accessibilityRole="button"`, rung nhẹ khi bấm không cần (bỏ qua).
- [ ] **Step 5: Run** `npm test && npm run typecheck` → PASS; `npx expo start` mở được trên Android emulator/Expo Go.
- [ ] **Step 6: Commit** `feat: scaffold app with large-text theme and BigButton`

---

### Task 2: Mở gọi điện / nhắn tin / Zalo / app khác

**Files:**
- Create: `src/features/launch/phone.ts`, `src/features/launch/launch.ts`
- Test: `src/features/launch/__tests__/phone.test.ts`, `src/features/launch/__tests__/launch.test.ts`

**Interfaces:**
- Produces:
  - `normalizePhone(raw: string): string | null` — về dạng nội địa chỉ chữ số (`0912345678`); `+84`/`84` đầu số di động → `0`; trả `null` nếu ít hơn 8 chữ số
  - `telUrl(phone: string): string`, `smsUrl(phone: string): string`, `zaloUrl(phone: string): string` (`https://zalo.me/<normalized>`)
  - `openUrlSafe(url: string): Promise<boolean>` — `Linking.openURL`, bắt lỗi trả `false`
  - `openApp(packageName: string): Promise<'opened' | 'store'>` — `IntentLauncher.openApplication`; lỗi → `Linking.openURL('market://details?id=' + pkg)`, nếu vẫn lỗi → `https://play.google.com/store/apps/details?id=` + pkg
  - `KNOWN_APPS: Record<'zalo' | 'vneid', { name: string; packageName: string }>`

- [ ] **Step 1: Write the failing tests**

```ts
test('normalizes messy numbers', () => {
  expect(normalizePhone('+84 912-345.678')).toBe('0912345678');
  expect(normalizePhone('84912345678')).toBe('0912345678');
  expect(normalizePhone('0912 345 678')).toBe('0912345678');
  expect(normalizePhone('(028) 3822 1234')).toBe('02838221234');
  expect(normalizePhone('113')).toBeNull();
});
test('builds urls', () => {
  expect(telUrl('+84912345678')).toBe('tel:0912345678');
  expect(smsUrl('0912345678')).toBe('sms:0912345678');
  expect(zaloUrl('+84 912 345 678')).toBe('https://zalo.me/0912345678');
});
test('openApp falls back to store', async () => {
  (IntentLauncher.openApplication as jest.Mock).mockRejectedValueOnce(new Error('not installed'));
  expect(await openApp('com.vnid')).toBe('store');
  expect(Linking.openURL).toHaveBeenCalledWith('market://details?id=com.vnid');
});
```

Lưu ý: số khẩn cấp ngắn (113/114/115) không đi qua `normalizePhone` — không có tính năng gọi số ngắn ở v1.

- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement**
- [ ] **Step 4: Run** `npm test` → PASS
- [ ] **Step 5: Commit** `feat: phone normalization and safe app/url launching`

---

### Task 3: Người thân (chọn từ danh bạ, gọi/nhắn/Zalo)

**Files:**
- Create: `src/features/people/peopleStore.ts`, `src/features/people/pickContact.ts`
- Create: `app/people/index.tsx` (danh sách), `app/people/[id].tsx` (3 nút), `app/people/add.tsx`
- Create: `src/ui/PermissionNotice.tsx`
- Test: `src/features/people/__tests__/peopleStore.test.ts`, `app/people/__tests__/add.test.tsx`, `app/people/__tests__/detail.test.tsx`

**Interfaces:**
- Consumes: `normalizePhone`, `telUrl`, `smsUrl`, `zaloUrl`, `openUrlSafe` (Task 2); `BigButton`, `Screen`, `AppText` (Task 1)
- Produces:
  - `type Person = { id: string; name: string; phone: string /* normalized */; photoUri?: string; isEmergency: boolean }`
  - `listPeople(): Promise<Person[]>`, `addPerson(p: Omit<Person, 'id' | 'isEmergency'>): Promise<Person>`, `removePerson(id: string): Promise<void>`, `setEmergency(id: string): Promise<void>` (chỉ một người là khẩn cấp), `getEmergency(): Promise<Person | null>` — khoá AsyncStorage `people.v1`
  - `pickContact(): Promise<{ status: 'ok'; name: string; phones: string[]; photoUri?: string } | { status: 'denied' } | { status: 'cancelled' }>` — `Contacts.requestPermissionsAsync` rồi `Contacts.presentContactPickerAsync`
  - `<PermissionNotice message: string />` — chữ to + BigButton "Mở cài đặt" gọi `Linking.openSettings()`

Màn hình chi tiết: ảnh/chữ cái đầu cỡ lớn, tên, 3 BigButton theo thứ tự **Gọi điện** (xanh lá) · **Gọi Zalo** (xanh Zalo) · **Nhắn tin** (xanh dương). Nút Xoá và "Đặt làm người gọi khẩn cấp" chỉ hiện khi `familyMode.enabled` (Task 7 cung cấp; trước Task 7 luôn hiện, Task 7 sẽ bọc lại). Nếu liên hệ có nhiều số, màn hình add cho chọn một số bằng các nút to.

- [ ] **Step 1: Write the failing tests**

```ts
test('add, list, set single emergency', async () => {
  const a = await addPerson({ name: 'Con Lan', phone: '0912345678' });
  const b = await addPerson({ name: 'Cháu Minh', phone: '0987654321' });
  await setEmergency(a.id); await setEmergency(b.id);
  expect((await getEmergency())?.id).toBe(b.id);
  expect((await listPeople()).filter(p => p.isEmergency)).toHaveLength(1);
});
test('shows settings button when permission denied', async () => {
  mockPickContact({ status: 'denied' });
  render(<AddPersonScreen />);
  fireEvent.press(screen.getByText('Chọn từ danh bạ'));
  expect(await screen.findByText('Mở cài đặt')).toBeTruthy();
});
test('detail Zalo button opens zalo.me link', async () => {
  seedPeople([{ id: '1', name: 'Con Lan', phone: '0912345678', isEmergency: false }]);
  render(<PersonDetailScreen id="1" />);
  fireEvent.press(await screen.findByText('Gọi Zalo'));
  expect(Linking.openURL).toHaveBeenCalledWith('https://zalo.me/0912345678');
});
```

- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement** store, picker, 3 màn hình. Khi `openUrlSafe` trả `false` → hiện "Không mở được. Bấm để thử lại" + nút thử lại.
- [ ] **Step 4: Run** `npm test && npm run typecheck` → PASS
- [ ] **Step 5: Commit** `feat: favourite people with call, SMS and Zalo buttons`

---

### Task 4: Kho Sổ tài khoản + khoá

**Files:**
- Create: `src/features/vault/vaultRepo.ts`, `src/features/vault/unlock.ts`, `src/features/settings/pin.ts`
- Test: `src/features/vault/__tests__/vaultRepo.test.ts`, `src/features/vault/__tests__/unlock.test.ts`, `src/features/settings/__tests__/pin.test.ts`

**Interfaces:**
- Produces:
  - `type VaultEntry = { id: string; appName: string; username: string; password: string; note: string; appKey?: 'vneid' | 'zalo' }`
  - `listEntries(): Promise<VaultEntry[]>`, `getEntry(id): Promise<VaultEntry | null>`, `saveEntry(e: Omit<VaultEntry,'id'> & { id?: string }): Promise<VaultEntry>`, `deleteEntry(id): Promise<void>` — SecureStore: chỉ mục `vault.index` (JSON mảng id), mỗi mục `vault.entry.<id>`
  - `validateEntry(e): string | null` — trả câu lỗi tiếng Việt: thiếu tên app; `note` > 500 ký tự; tổng JSON > 2000 byte UTF-8
  - `hasPin(): Promise<boolean>`, `setPin(pin: string): Promise<void>` (đúng 4 chữ số, ngược lại throw), `checkPin(pin: string): Promise<boolean>` — SecureStore `settings.pin`
  - `unlockVault(askPin: () => Promise<string | null>): Promise<boolean>` — nếu `LocalAuthentication.getEnrolledLevelAsync()` > `NONE`: `authenticateAsync({ promptMessage: 'Mở Sổ tài khoản', cancelLabel: 'Huỷ' })`; nếu `NONE`: gọi `askPin()` rồi `checkPin`

- [ ] **Step 1: Write the failing tests**

```ts
test('round-trips Vietnamese text and keeps index', async () => {
  const e = await saveEntry({ appName: 'VNeID', username: '012345678901', password: 'Mật@khẩu1', note: 'Số CCCD của bà', appKey: 'vneid' });
  expect(await getEntry(e.id)).toEqual(e);
  expect((await listEntries()).map(x => x.id)).toEqual([e.id]);
  await deleteEntry(e.id);
  expect(await listEntries()).toEqual([]);
});
test('rejects oversized note', () => {
  expect(validateEntry({ appName: 'A', username: '', password: '', note: 'x'.repeat(501) })).toMatch(/quá dài/);
});
test('setPin rejects non 4-digit', async () => {
  await expect(setPin('12a4')).rejects.toThrow();
  await setPin('2580'); expect(await checkPin('2580')).toBe(true); expect(await checkPin('0000')).toBe(false);
});
test('falls back to app PIN when device has no security', async () => {
  mockEnrolledLevel(SecurityLevel.NONE); await setPin('2580');
  expect(await unlockVault(async () => '2580')).toBe(true);
  expect(LocalAuthentication.authenticateAsync).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement**
- [ ] **Step 4: Run** → PASS
- [ ] **Step 5: Commit** `feat: encrypted on-device account vault with device/PIN unlock`

---

### Task 5: Màn hình Sổ tài khoản

**Files:**
- Create: `app/vault/index.tsx` (khoá → danh sách), `app/vault/[id].tsx` (xem), `app/vault/edit.tsx` (thêm/sửa), `src/ui/PinPad.tsx`
- Test: `app/vault/__tests__/vault.test.tsx`

**Interfaces:**
- Consumes: vault API + `unlockVault` (Task 4), `openApp`, `KNOWN_APPS` (Task 2)
- Produces: `<PinPad title: string; onDone(pin: string): void; onCancel(): void />` — 10 phím số to + Xoá, dùng lại ở Task 7

Hành vi:
- Vào `/vault` → gọi `unlockVault` ngay; thất bại → "Chưa mở được sổ" + nút "Thử lại". Sổ tự khoá lại khi app xuống nền (`AppState`).
- Màn xem: tên app (title), "Tên đăng nhập" và "Mật khẩu" chữ cỡ `title`, mật khẩu hiện sẵn (người dùng là ông bà, đã qua khoá); mỗi dòng có nút "Sao chép" (`expo-clipboard`, hiện "Đã sao chép" 2 giây). Nếu `appKey` có → BigButton "Mở VNeID"/"Mở Zalo". Nút "Xem hướng dẫn quên mật khẩu" khi `appKey === 'vneid'` → `/guides/quen-mat-khau-vneid`.
- Thêm/sửa/xoá chỉ hiện khi family mode (bọc ở Task 7). Form edit: chọn nhanh "VNeID" / "Zalo" / "Khác" bằng nút to, hiện lỗi `validateEntry`.

- [ ] **Step 1: Write the failing tests**

```tsx
test('locked vault shows retry when unlock fails', async () => {
  mockUnlock(false);
  render(<VaultIndex />);
  expect(await screen.findByText('Thử lại')).toBeTruthy();
});
test('entry screen copies password and opens VNeID', async () => {
  mockUnlock(true); const e = await saveEntry({ appName: 'VNeID', username: 'u', password: 'p@1', note: '', appKey: 'vneid' });
  render(<VaultEntryScreen id={e.id} />);
  fireEvent.press((await screen.findAllByText('Sao chép'))[1]);
  expect(Clipboard.setStringAsync).toHaveBeenCalledWith('p@1');
  fireEvent.press(screen.getByText('Mở VNeID'));
  expect(IntentLauncher.openApplication).toHaveBeenCalledWith('com.vnid');
});
```

- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement**
- [ ] **Step 4: Run** → PASS
- [ ] **Step 5: Commit** `feat: account vault screens`

---

### Task 6: Hướng dẫn (Supabase + offline + đọc to)

**Files:**
- Create: `supabase/migrations/20261025000000_guides.sql`, `supabase/seed.sql`
- Create: `src/features/guides/types.ts`, `src/features/guides/seedGuides.ts`, `src/features/guides/supabase.ts`, `src/features/guides/guidesRepo.ts`, `src/features/guides/speak.ts`
- Create: `app/guides/index.tsx`, `app/guides/[slug].tsx`
- Create: `.env.example`
- Test: `src/features/guides/__tests__/guidesRepo.test.ts`, `app/guides/__tests__/guide.test.tsx`

**Interfaces:**
- Produces:
  - `type GuideStep = { text: string; imageUrl?: string }`; `type Guide = { slug: string; title: string; icon: string; sort: number; steps: GuideStep[]; updatedAt: string }`
  - `SEED_GUIDES: Guide[]` — ít nhất 6 bài: `dang-nhap-vneid`, `quen-mat-khau-vneid`, `goi-zalo`, `goi-dien`, `nhan-tin`, `mo-danh-ba` (3–7 bước mỗi bài, viết câu ngắn, xưng "ông/bà")
  - `getGuides(fetchRemote?: () => Promise<Guide[]>): Promise<Guide[]>` — remote (timeout 5 s) → ghi cache `guides.cache.v1` → trả về; lỗi → cache; không có cache → `SEED_GUIDES`. Luôn sắp theo `sort`.
  - `getGuide(slug: string): Promise<Guide | null>` — tìm trong kết quả `getGuides`
  - `speak(text: string): void` / `stopSpeaking(): void` — `Speech.speak(text, { language: 'vi-VN', rate: 0.85 })`

SQL: bảng `public.guides(slug text primary key, title text not null, icon text not null, sort int not null default 0, steps jsonb not null, published boolean not null default true, updated_at timestamptz not null default now())`; `enable row level security`; policy `select` cho `anon` với `using (published)`; không có policy ghi. Nội dung được thêm qua Supabase Studio. `seed.sql` chèn cùng 6 bài như `SEED_GUIDES`.

Màn chi tiết: mỗi lần một bước ("Bước 2/5"), chữ cỡ `large`, ảnh nếu có, 3 nút to: "◀ Trước" · "🔊 Đọc to" · "Tiếp ▶". Bước cuối: "Xong" quay về danh sách. Rời màn → `stopSpeaking()`.

- [ ] **Step 1: Write the failing tests**

```ts
test('returns remote and caches it', async () => {
  const remote = [guide('b', 2), guide('a', 1)];
  expect((await getGuides(async () => remote)).map(g => g.slug)).toEqual(['a', 'b']);
  expect((await getGuides(async () => { throw new Error('offline'); })).map(g => g.slug)).toEqual(['a', 'b']);
});
test('returns seed when offline and no cache', async () => {
  expect(await getGuides(async () => { throw new Error('offline'); })).toEqual(SEED_GUIDES);
});
test('guide screen steps forward and speaks', async () => {
  render(<GuideScreen slug="goi-zalo" />);
  expect(await screen.findByText(/Bước 1\//)).toBeTruthy();
  fireEvent.press(screen.getByText(/Đọc to/));
  expect(Speech.speak).toHaveBeenCalledWith(expect.any(String), expect.objectContaining({ language: 'vi-VN' }));
  fireEvent.press(screen.getByText(/Tiếp/));
  expect(screen.getByText(/Bước 2\//)).toBeTruthy();
});
```

- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement**; chạy migration + seed trên project Supabase có sẵn (SQL Editor trong dashboard hoặc `supabase db push`), điền `.env` (không commit `.env`).
- [ ] **Step 4: Run** `npm test` → PASS; trên máy thật bật chế độ máy bay, mở app lần đầu → thấy 6 bài mẫu.
- [ ] **Step 5: Commit** `feat: offline-first step-by-step guides with Vietnamese TTS`

---

### Task 7: Màn hình chính, Gọi khẩn cấp, Chế độ cài đặt

**Files:**
- Modify: `app/index.tsx` (4 ô lớn dạng lưới 2×2), `app/_layout.tsx` (bọc `FamilyModeProvider`)
- Create: `src/features/settings/FamilyModeContext.tsx`, `app/settings.tsx`, `app/setup.tsx` (lần đầu mở app)
- Modify: `app/people/[id].tsx`, `app/people/index.tsx`, `app/vault/*.tsx` — ẩn nút sửa/xoá/thêm khi không ở family mode
- Test: `app/__tests__/home.test.tsx`, `src/features/settings/__tests__/familyMode.test.tsx`

**Interfaces:**
- Consumes: `getEmergency`, `telUrl`, `openUrlSafe`, `PinPad`, `hasPin/setPin/checkPin`, `useSettings`
- Produces: `useFamilyMode(): { enabled: boolean; unlock(pin: string): Promise<boolean>; lock(): void }` — tự `lock()` khi `AppState` sang `background`

Màn chính: ô **Người thân**, **Sổ tài khoản**, **Hướng dẫn**, **Gọi khẩn cấp** (đỏ). Gọi khẩn cấp: có người khẩn cấp → mở `tel:` ngay; chưa có → "Chưa chọn người gọi khẩn cấp. Nhờ con cháu cài đặt." Góc dưới có nút nhỏ hơn (vẫn ≥ 56) "Cài đặt (cho con cháu)" → PinPad → `/settings`.

Lần đầu mở (`hasPin() === false`): `/setup` — 3 bước: đặt PIN 4 số (nhập 2 lần), chọn cỡ chữ, thêm người thân đầu tiên (có thể bỏ qua).

`/settings`: chọn cỡ chữ (3 nút xem trước chữ "Aa"), đổi PIN, các lối tắt "Thêm người thân", "Thêm tài khoản", nút "Xong" → `lock()`.

- [ ] **Step 1: Write the failing tests**

```tsx
test('emergency tile calls emergency contact', async () => {
  seedPeople([{ id: '1', name: 'Con Lan', phone: '0912345678', isEmergency: true }]);
  render(<Home />);
  fireEvent.press(await screen.findByText('Gọi khẩn cấp'));
  await waitFor(() => expect(Linking.openURL).toHaveBeenCalledWith('tel:0912345678'));
});
test('emergency tile explains when none set', async () => {
  render(<Home />);
  fireEvent.press(await screen.findByText('Gọi khẩn cấp'));
  expect(await screen.findByText(/Chưa chọn người gọi khẩn cấp/)).toBeTruthy();
});
test('family mode unlocks with right PIN and locks on background', async () => {
  await setPin('2580');
  const { result } = renderHook(useFamilyMode, { wrapper: FamilyModeProvider });
  expect(await act(() => result.current.unlock('1111'))).toBe(false);
  await act(() => result.current.unlock('2580')); expect(result.current.enabled).toBe(true);
  act(() => emitAppState('background')); expect(result.current.enabled).toBe(false);
});
```

- [ ] **Step 2: Run** → FAIL
- [ ] **Step 3: Implement**
- [ ] **Step 4: Run** `npm test && npm run typecheck` → PASS
- [ ] **Step 5: Commit** `feat: home grid, emergency call, PIN-protected family setup mode`

---

### Task 8: Chuẩn bị phát hành CH Play

**Files:**
- Modify: `app.json`
- Create: `eas.json`, `docs/privacy-policy.md`, `docs/store-listing.md`, `assets/icon.png`, `assets/adaptive-icon.png`, `assets/splash.png`

**Interfaces:** không có code mới cho task khác.

- [ ] **Step 1: `app.json`** — `name: "Trợ Lý Ông Bà"`, `slug: "tro-ly-ong-ba"`, `android.package: "vn.trolyongba.app"`, `android.versionCode: 1`, `android.permissions: ["READ_CONTACTS", "USE_BIOMETRIC"]`, `android.blockedPermissions: ["android.permission.SEND_SMS", "android.permission.CALL_PHONE", "android.permission.READ_CALL_LOG", "android.permission.RECORD_AUDIO"]`, `android.allowBackup: false`, plugin `expo-secure-store`, `expo-local-authentication` (`faceIDPermission` bỏ), `expo-contacts` với câu xin quyền tiếng Việt: "Để ông bà chọn người thân từ danh bạ".
- [ ] **Step 2: Kiểm tra quyền thực tế** — `npx expo prebuild --platform android --clean` rồi kiểm `android/app/src/main/AndroidManifest.xml`: không có SEND_SMS/CALL_PHONE/READ_CALL_LOG. Xoá thư mục `android/` sau khi kiểm (dùng CNG).
- [ ] **Step 3: `eas.json`** — profile `preview` (`android.buildType: "apk"`), `production` (`aab`, `autoIncrement: true`); env Supabase lấy từ EAS secrets.
- [ ] **Step 4: Privacy policy + store listing** — nêu rõ: danh bạ chỉ đọc khi người dùng chọn, lưu trên máy; mật khẩu mã hoá trên máy, không gửi đi; không thu thập dữ liệu cá nhân; chỉ tải nội dung hướng dẫn. Đăng privacy policy lên một URL công khai (GitHub Pages hoặc Notion) và điền vào Play Console. Data safety form: "Không thu thập, không chia sẻ".
- [ ] **Step 5: Build thử** `eas build -p android --profile preview` → cài APK lên máy Android thật, chạy checklist: gọi điện, nhắn tin, Gọi Zalo (có/không cài Zalo), mở VNeID (xác minh package `com.vnid`; nếu sai sửa `KNOWN_APPS`), Sổ tài khoản với vân tay và với máy không khoá, hướng dẫn offline, đọc to tiếng Việt (nếu máy thiếu giọng vi-VN, hướng dẫn cài trong Cài đặt → Chuyển văn bản thành giọng nói), cỡ chữ Cực lớn không bị tràn chữ.
- [ ] **Step 6: Build production** `eas build -p android --profile production` → upload AAB lên Play Console, track **Internal testing** trước (tài khoản cá nhân mới cần closed testing 12 người × 14 ngày trước khi lên production).
- [ ] **Step 7: Commit** `chore: Play Store release configuration`
