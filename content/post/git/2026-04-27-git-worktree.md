---
title: Git Worktree
author: phongthien99
date: 2026-04-27 12:40:00 +0800
categories: [develop]
tags: [develop]
math: true
media_subpath: '/posts/20240503'
---

# Git Worktree 

> *Bạn đang code ngon lành thì sếp ping "fix gấp bug production". Bạn thở dài, `git stash`, `checkout`, fix xong lại `checkout` về, `stash pop`... rồi mất 10 phút nhớ lại mình đang làm gì. Quen không?*


## 1. Đặt vấn đề — Context Switch là kẻ giết năng suất thầm lặng

Hãy hình dung ngày làm việc điển hình của một developer:

Bạn đang triển khai feature **"Payment Integration"** — đã setup xong service, viết được nửa logic xử lý, server dev đang chạy ngon lành trên `localhost:3000`. Đúng lúc đó, Slack nổ: *"Production đang lỗi checkout, cần hotfix ngay!"*.

Và bạn bắt đầu vòng lặp quen thuộc:

```bash
git stash                          # Cất đống code đang làm dở
git checkout main                  # Về nhánh chính
git pull origin main               # Cập nhật code mới nhất
git checkout -b hotfix/checkout    # Tạo nhánh hotfix
# ... fix bug, test, commit, push ...
git checkout feature/payment       # Quay lại feature
git stash pop                      # Lấy code cũ ra
# ... 10 phút sau: "Mình đang làm tới đâu nhỉ?" 🤯
```

Vấn đề không chỉ dừng ở đó. Có ít nhất **ba nỗi đau** mà developer gặp hàng ngày:

**Nỗi đau #1 — Hotfix gấp giữa chừng feature.** Như kịch bản trên. `git stash` nghe đơn giản nhưng thực tế đầy rủi ro: stash conflict, quên stash nào là stash nào, hoặc tệ hơn — `stash pop` nhầm branch.

**Nỗi đau #2 — Review code của đồng nghiệp.** Bạn đồng nghiệp gửi PR và cần bạn pull branch về test thử. Nhưng branch hiện tại của bạn đang dở dang, `node_modules` đang chạy, dev server đang bật. Checkout sang branch khác đồng nghĩa với việc phải tắt server, có thể cài lại dependencies, rồi lại quay về setup lại mọi thứ.

**Nỗi đau #3 — AI agent cần isolation.** Trong kỷ nguyên AI-assisted development, bạn muốn cho Claude Code hoặc Copilot Workspace chạy task trên một branch riêng, trong khi bạn vẫn code trên branch của mình. Nhưng cả hai đều cần working directory — và bạn chỉ có một.

Tất cả những vấn đề này có chung một gốc rễ: **Git chỉ cho phép một branch hoạt động trong một working directory tại một thời điểm.** Và giải pháp không phải là clone thêm repo, mà là một tính năng đã có sẵn trong Git từ lâu nhưng ít người biết đến.



## 2. Giải pháp — Git Worktree là gì?

**Git Worktree** cho phép bạn checkout nhiều branch ra nhiều thư mục khác nhau, tất cả đều dùng chung một `.git` directory duy nhất.

Nói đơn giản: thay vì chỉ có một "bản sao làm việc" của repo, bạn có thể mở ra nhiều bản sao, mỗi bản ở một branch khác nhau, trong các thư mục riêng biệt — mà không cần `git clone` thêm lần nào.

### Cách nó hoạt động bên trong

Khi bạn `git clone` một repo, Git tạo ra hai thứ: thư mục `.git` (chứa toàn bộ object database, refs, config) và working tree (các file bạn nhìn thấy và edit). Bình thường, chúng là cặp 1:1. Git Worktree phá vỡ giới hạn đó — bạn có thể có nhiều working tree nhưng tất cả trỏ về cùng một `.git`.

```
~/my-project/              ← worktree chính (main branch)
├── .git/                  ← object database duy nhất
├── src/
└── package.json

/tmp/hotfix-wt/            ← worktree phụ (hotfix branch)
├── .git  → file trỏ về ~/my-project/.git
├── src/
└── package.json

/tmp/review-wt/            ← worktree phụ (PR review branch)
├── .git  → file trỏ về ~/my-project/.git
├── src/
└── package.json
```




## 3. Các lệnh cơ bản

### Tạo worktree mới

```bash
# Tạo worktree mới từ branch có sẵn
git worktree add ../hotfix-wt hotfix/urgent-bug

# Tạo worktree + tạo branch mới (flag -b)
git worktree add ../hotfix-wt -b hotfix/urgent-bug

# Tạo worktree từ remote branch
git worktree add ../review-wt origin/feature/new-ui
```

### Xem danh sách worktree

```bash
git worktree list
# Output:
# /home/dev/my-project         abc1234 [main]
# /home/dev/hotfix-wt          def5678 [hotfix/urgent-bug]
# /home/dev/review-wt          ghi9012 [feature/new-ui]
```

### Xóa worktree

```bash
# Xóa worktree (sẽ xóa cả thư mục)
git worktree remove ../hotfix-wt

# Nếu có uncommitted changes, cần force
git worktree remove --force ../hotfix-wt

# Dọn dẹp metadata của worktree đã xóa thủ công
git worktree prune
```

### Quy tắc quan trọng

Một branch chỉ được checkout ở đúng một worktree tại một thời điểm. Nếu `main` đang được dùng ở worktree chính, bạn không thể tạo worktree khác cũng checkout `main`. Đây là thiết kế có chủ đích để tránh xung đột khi commit.



## 4. Workflow thực tế

### Workflow 1 — Hotfix song song với feature đang làm dở

Đây là use case kinh điển nhất. Bạn đang code feature, cần fix bug production ngay mà không muốn đụng đến bất cứ thứ gì đang chạy.

```bash
# Bạn đang ở ~/project, branch feature/payment, server đang chạy

# Tạo worktree cho hotfix (không ảnh hưởng gì tới thư mục hiện tại)
git worktree add ../hotfix-checkout -b hotfix/fix-checkout origin/main

# Mở terminal mới, cd vào worktree
cd ../hotfix-checkout
npm install          # Cài dependencies riêng cho worktree này
npm run dev          # Chạy dev server trên port khác

# Fix bug, test, commit
git add .
git commit -m "fix: resolve checkout payment error"
git push origin hotfix/fix-checkout

# Tạo PR, merge xong → dọn dẹp
cd ~/project
git worktree remove ../hotfix-checkout
```

Trong suốt quá trình này, terminal cũ của bạn vẫn chạy feature/payment, dev server vẫn hoạt động, code vẫn nguyên vẹn. Không stash, không checkout, không mất context.

### Workflow 2 — Review PR của đồng nghiệp

Đồng nghiệp gửi PR cần bạn test thử trên máy local trước khi approve. Thay vì phá working directory hiện tại, bạn tạo worktree riêng.

```bash
# Fetch branch mới nhất
git fetch origin

# Tạo worktree cho branch cần review
git worktree add ../review-new-ui origin/feature/new-ui

# Mở thư mục này trong VSCode
code ../review-new-ui

# Test, đọc code, chạy thử
cd ../review-new-ui
npm install
npm test
npm run dev

# Review xong → dọn dẹp
git worktree remove ../review-new-ui
```

Bạn thậm chí có thể mở VSCode **Multi-root Workspace** để thấy cả hai codebase cùng lúc:

```bash
# Mở workspace với nhiều worktree
code --add ../review-new-ui
# Hoặc tạo file .code-workspace
```

```json
// multi-root.code-workspace
{
  "folders": [
    { "path": "../project", "name": "🔨 Feature - Payment" },
    { "path": "../review-new-ui", "name": "👀 Review - New UI" }
  ]
}
```

### Workflow 3 — Phát triển nhiều feature song song

Khi team nhỏ, một developer thường phải xử lý 2–3 feature cùng lúc. Feature A đang chờ review, bạn chuyển sang feature B. Feature B bị block bởi API chưa ready, bạn nhảy sang feature C. Với worktree, mỗi feature là một thư mục riêng.

```bash
# Setup ban đầu
git worktree add ../ft-payment   -b feature/payment    origin/main
git worktree add ../ft-dashboard -b feature/dashboard   origin/main
git worktree add ../ft-auth      -b feature/auth        origin/main

# Mỗi worktree chạy trên terminal/IDE riêng
# Terminal 1: cd ../ft-payment   && npm i && npm run dev -- --port 3001
# Terminal 2: cd ../ft-dashboard && npm i && npm run dev -- --port 3002
# Terminal 3: cd ../ft-auth      && npm i && npm run dev -- --port 3003

# Nhảy qua lại giữa các feature = Alt+Tab giữa các terminal
# Không stash, không checkout, không mất state
```

Mẹo quản lý: đặt tên thư mục worktree có prefix để dễ nhận diện, ví dụ `ft-` cho feature, `hf-` cho hotfix, `rv-` cho review. Chạy `git worktree list` để luôn nắm toàn cảnh.

### Workflow 4 — Merge và giải quyết conflict an toàn

Merge conflict luôn là một thao tác căng thẳng — đặc biệt khi bạn merge trực tiếp trên branch đang làm việc. Với worktree, bạn có thể tạo một "sandbox" riêng để merge.

```bash
# Tạo worktree riêng để merge
git worktree add ../merge-zone -b merge/payment-to-main origin/main

cd ../merge-zone

# Merge feature branch vào
git merge origin/feature/payment

# Nếu có conflict → giải quyết thoải mái ở đây
# Code trong worktree chính không bị ảnh hưởng
# Bạn có thể mở cả hai thư mục so sánh song song

# Merge xong, test kỹ
npm install
npm test

# Push và tạo PR
git push origin merge/payment-to-main

# Dọn dẹp
cd ~/project
git worktree remove ../merge-zone
```

Lợi ích lớn nhất: nếu merge hỏng hoặc bạn muốn làm lại, chỉ cần xóa worktree và tạo mới. Không có gì bị ảnh hưởng.

### Workflow 5 — AI Agent chạy song song (Claude Code, Copilot)

Đây là workflow đang ngày càng phổ biến. Bạn muốn một AI agent tự động implement một task trong khi bạn làm task khác — hoặc thậm chí nhiều agent chạy song song nhiều task.

```bash
# Worktree cho bạn
# (đang ở ~/project, branch feature/payment)

# Worktree cho AI agent 1
git worktree add ../agent-1-auth -b feature/auth origin/main

# Worktree cho AI agent 2
git worktree add ../agent-2-tests -b chore/add-tests origin/main

# Chạy Claude Code trên từng worktree riêng biệt
# Terminal 1 (agent 1):
cd ../agent-1-auth
claude "Implement OAuth2 login flow based on the auth spec in docs/"

# Terminal 2 (agent 2):
cd ../agent-2-tests
claude "Write unit tests for all services in src/services/"

# Bạn vẫn code bình thường ở ~/project
# Mỗi agent có sandbox riêng, không đụng chạm nhau
```

Kiến trúc này đặc biệt mạnh vì mỗi agent có **full isolation**: filesystem riêng, có thể cài dependencies riêng, chạy test riêng, commit riêng — không sợ race condition hay file conflict.

### Workflow 6 — Test trên nhiều version/config cùng lúc

Bạn cần test ứng dụng với các config khác nhau — ví dụ Node 18 vs Node 20, hoặc PostgreSQL vs MySQL. Worktree cho phép bạn chạy song song.

```bash
git worktree add ../test-node18 main
git worktree add ../test-node20 main  
# Lưu ý: cùng branch main sẽ bị lỗi!
# Giải pháp: checkout dạng detached HEAD
git worktree add --detach ../test-node20 main
```

```bash
# Terminal 1
cd ../test-node18
nvm use 18
npm install && npm test

# Terminal 2
cd ../test-node20
nvm use 20
npm install && npm test
```



## 5. Cheat Sheet — Lệnh thường dùng

```bash
# ─── TẠO ──────────────────────────────────────────────
git worktree add <path> <branch>         # Checkout branch có sẵn
git worktree add <path> -b <new> <base>  # Tạo branch mới từ base
git worktree add --detach <path> <ref>   # Checkout dạng detached HEAD

# ─── QUẢN LÝ ──────────────────────────────────────────
git worktree list                        # Liệt kê tất cả worktree
git worktree list --porcelain            # Output dạng machine-readable

# ─── DỌN DẸP ──────────────────────────────────────────
git worktree remove <path>               # Xóa worktree
git worktree remove --force <path>       # Xóa kể cả khi có changes
git worktree prune                       # Dọn metadata cũ

# ─── MẸO ──────────────────────────────────────────────
git worktree lock <path>                 # Khóa worktree (tránh bị prune)
git worktree unlock <path>               # Mở khóa
git worktree move <path> <new-path>      # Di chuyển worktree
```

---

## 6. Kết luận

Git Worktree không phải tính năng mới — nó đã có trong Git từ version 2.5 (2015). Nhưng nó là một trong những công cụ bị underrated nhất trong bộ công cụ của developer.

Chỉ với vài lệnh đơn giản, bạn giải quyết được những vấn đề mà trước đây phải dùng `stash`, clone repo, hoặc chấp nhận mất context. Hotfix không còn là nỗi ám ảnh. Review PR trở nên nhẹ nhàng. Phát triển song song nhiều feature trở thành chuyện bình thường.

Và đặc biệt, trong kỷ nguyên AI agent, khi bạn muốn Claude Code hay bất kỳ AI tool nào chạy task song song với bạn — Git Worktree cung cấp isolation hoàn hảo mà không cần container hay VM phức tạp.

Hãy thử ngay hôm nay. Chỉ cần một lệnh `git worktree add` — và bạn sẽ tự hỏi tại sao mình không biết đến nó sớm hơn.

