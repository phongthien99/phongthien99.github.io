---
title: "Feature-Centric Architecture with Clean Principle"
author: phongthien99
date: 2025-11-08 11:09:00 +0700
categories: [Fe]
tags: [nginx,fe,nextjs,qiankun,micro-frontend]
math: true
media_subpath: '/posts/20251026'

---
# Feature-Centric Architecture with Clean Principle
> Hướng dẫn tổ chức code theo feature với Clean Architecture để xây dựng ứng dụng dễ bảo trì và mở rộng
> 


## 1. Đặt vấn đề

Trong các dự án React/Next.js truyền thống, code thường được tổ chức theo **technical layers** như `components/`, `hooks/`, `services/`, `types/`, `utils/`. Mỗi folder gom tất cả file cùng loại, dẫn đến nhiều vấn đề khi dự án lớn:

---

- **Khó tìm kiếm và điều hướng:** Muốn thay đổi một feature, developer phải nhảy qua nhiều folder (UI, logic, API, types, utils), mất thời gian và khó nắm tổng thể.
- **Dependencies rối rắm:** Component import từ nhiều layer, khó xác định ảnh hưởng khi thay đổi, dễ circular dependency và test phức tạp.
- **Code duplication & inconsistency:** Logic giống nhau nhưng khác nhau giữa các component, dẫn đến bug, tốn công sửa và test.
- **Coupling giữa features:** Feature này phụ thuộc nhiều feature khác, khó tách riêng, khó reuse và tree-shaking kém hiệu quả.
- **Khó mở rộng & parallel development:** Service chứa tất cả logic domain dẫn đến Git conflict, khó phân công task, file lớn gây khó review.

Để khắc phục, cần **feature-based organization**, tức tổ chức code theo từng feature thay vì loại file. Cách này giúp code **liên quan gần nhau, độc lập, dễ tìm, dễ test, mở rộng nhanh và giảm bugs**, cải thiện đáng kể maintainability và khả năng phát triển song song.

## 2. Giải pháp

### 2.1. Feature-Based Organization với Clean Architecture

Thay vì tổ chức theo **technical layers**, chúng ta tổ chức theo **business features** kết hợp với **Clean Architecture principles**:

```
src/app/
├── login/                    # ✅ FEATURE: Login
│   ├── components/           # UI của feature
│   ├── hooks/               # Application logic
│   ├── context/             # Shared context
│   ├── entities/            # Domain models
│   ├── dto/                 # Data transfer objects
│   ├── interfaces/          # Contracts
│   ├── usecases/            # Business logic
│   ├── repositories/        # Data access
│   ├── providers/           # Infrastructure
│   └── page.tsx             # Entry point
│
├── signup/                   # ✅ FEATURE: Signup

```

### 2.2. Kiến Trúc Tổng Quan của mỗi  feature
{{< mermaid >}}
graph TB
    subgraph "Presentation Layer"
        components[components/]
        pages[page.tsx]
        providers[providers/]
    end
    
    subgraph "Application Layer"
        hooks[hooks/]
    end
    
    subgraph "Shared Context Layer"
        context[context/]
    end
    
    subgraph "Domain Layer"
        entities[entities/]
        dto[dto/]
        interfaces[interfaces/]
        usecases[usecases/]
    end
    
    subgraph "Infrastructure Layer"
        repositories[repositories/]
        
    end
    
    pages --> providers
    pages --> components
    
    components --> hooks
    components --> usecases
    
    hooks --> context
    providers --> context
    
    context --> interfaces
    
    hooks --> entities
    hooks --> dto
    
    repositories --> interfaces
    repositories --> entities
    repositories --> dto
    
    providers --> repositories
    
    usecases --> dto
    
    style context fill:#FF1493,stroke:#C71585,stroke-width:4px
    style entities fill:#90EE90,stroke:#2ecc71,stroke-width:2px
    style dto fill:#90EE90,stroke:#2ecc71,stroke-width:2px
    style interfaces fill:#90EE90,stroke:#2ecc71,stroke-width:2px
    style usecases fill:#90EE90,stroke:#2ecc71,stroke-width:2px
    style repositories fill:#FFD700,stroke:#f39c12,stroke-width:2px
    style providers fill:#FFD700,stroke:#f39c12,stroke-width:2px
    style hooks fill:#DDA0DD,stroke:#9b59b6,stroke-width:2px
    style components fill:#74b9ff,stroke:#3498db,stroke-width:2px
    style pages fill:#74b9ff,stroke:#3498db,stroke-width:2px

{{< /mermaid >}}
### 2.3. Nguyên Tắc Thiết Kế

### **Nguyên tắc 1: Feature Isolation**

```tsx
// ✅ Mỗi feature là self-contained
src/app/login/
  - Có đầy đủ layers riêng
  - Không phụ thuộc vào features khác
  - Có thể extract thành package độc lập

```

### **Nguyên tắc 2: Layer Dependency Rule**

```
Presentation → Application → Domain ← Infrastructure
                                ↑
                          Không phụ thuộc
                          vào outer layers

```

### **Nguyên tắc 3: Context Layer cho DI**

```tsx

hooks/ ──────┐
             ├──→ context/ ──→ interfaces/
providers/ ──┘

```

---

## 3. Thực hiện

### 3.1. Cấu Trúc Chi Tiết Một Feature

### **Example: Login Feature**

```
src/app/login/
│
├── components/                   # PRESENTATION LAYER
│   ├── LoginForm.tsx            # Main form component
│   ├── LoginButton.tsx          # Reusable button
│   └── SocialLoginButtons.tsx   # Social auth buttons
│
├── hooks/                        # APPLICATION LAYER
│   ├── UseLogin.ts              # Login logic hook
│   ├── useAuthRepository.ts     # Repository access hook
│   └── useLoginValidation.ts    # Validation hook
│
├── context/                      # SHARED CONTEXT LAYER
│   ├── AuthContext.tsx          # Context definition
│   └── index.ts                 # Exports
│
├── entities/                     # DOMAIN LAYER - Models
│   ├── User.ts                  # User entity
│   └── AuthSession.ts           # Session entity
│
├── dto/                          # DOMAIN LAYER - DTOs
│   └── LoginTypes.ts            # Request/Response types
│
├── interfaces/                   # DOMAIN LAYER - Contracts
│   ├── IAuthRepository.ts       # Repository interface
│   ├── IAuthenticator.ts        # Authenticator interface
│   ├── ISessionManager.ts       # Session manager interface
│   └── index.ts                 # Exports
│
├── usecases/                     # DOMAIN LAYER - Business Logic
│   └── LoginLogic.ts            # Pure business functions
│
├── repositories/                 # INFRASTRUCTURE LAYER - Data Access
│   ├── ApiAuthRepository.ts     # API implementation
│   └── LocalStorageAuthRepository.ts  # LocalStorage implementation
│
├── providers/                    # INFRASTRUCTURE LAYER - DI
│   └── AuthProvider.tsx         # Context provider
│
├── page.tsx                      # Route entry point
├── index.ts                      # Public API exports
└── README.md                     # Feature documentation

```


### 3.2. Implementation Examples

### **3.2.1. Context Layer**

```tsx
// context/AuthContext.tsx
"use client";

import { createContext } from "react";
import type { IAuthRepository } from "../interfaces/IAuthRepository";

/**
 * Auth Context - Shared between providers and hooks
 * Tách biệt coupling giữa Infrastructure và Application layers
 */
export const AuthContext = createContext<IAuthRepository | null>(null);

export type AuthContextValue = IAuthRepository | null;

```

### **3.2.2. Domain Layer - Entities**

```tsx
// entities/AuthSession.ts
/**
 * AuthSession Entity
 * Rich domain model với business logic
 */
export class AuthSession {
  constructor(
    public readonly token: string,
    public readonly user: User,
    public readonly expiresAt: Date,
    public readonly refreshToken?: string,
  ) {}

  /**
   * Business logic: Check if expired
   */
  isExpired(): boolean {
    return new Date() > this.expiresAt;
  }

  /**
   * Business logic: Check if needs refresh
   */
  needsRefresh(): boolean {
    const thirtyMinutes = 30 * 60 * 1000;
    return (this.expiresAt.getTime() - Date.now()) < thirtyMinutes;
  }

  /**
   * Persistence helper
   */
  save(): void {
    if (typeof window === 'undefined') return;
    localStorage.setItem('auth_session', JSON.stringify(this.serialize()));
  }

  private serialize() {
    return {
      token: this.token,
      user: { id: this.user.id, email: this.user.email, name: this.user.name },
      expiresAt: this.expiresAt.toISOString(),
      refreshToken: this.refreshToken,
    };
  }

  /**
   * Factory method
   */
  static fromResponse(data: any): AuthSession {
    const user = new User(data.user.id, data.user.email, data.user.name);
    return new AuthSession(
      data.token,
      user,
      new Date(data.expiresAt),
      data.refreshToken
    );
  }
}

```

**✨ Lợi ích:**

- ✅ Encapsulate business logic trong entity
- ✅ Self-validating, self-contained
- ✅ Dễ test (pure logic)
- ✅ Tái sử dụng trong toàn feature

### **3.2.3. Infrastructure Layer - Repositories**

```tsx
// repositories/ApiAuthRepository.ts
import { IAuthRepository } from "../interfaces/IAuthRepository";
import { LoginInput } from "../dto/LoginTypes";
import { AuthSession } from "../entities/AuthSession";

/**
 * API Implementation of AuthRepository
 * Isolated data access logic
 */
export class ApiAuthRepository implements IAuthRepository {
  constructor(private readonly baseUrl: string = "/api/auth") {}

  async login(input: LoginInput): Promise<AuthSession> {
    const res = await fetch(`${this.baseUrl}/login`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(input),
    });

    if (!res.ok) {
      throw new Error("Login failed");
    }

    const data = await res.json();
    return AuthSession.fromResponse(data);
  }

  async logout(): Promise<void> {
    await fetch(`${this.baseUrl}/logout`, { method: "POST" });
    AuthSession.clear();
  }

  getCurrentSession(): AuthSession | null {
    return AuthSession.load();
  }
}

```

**✨ Lợi ích:**

- ✅ Implement interface từ Domain
- ✅ Dễ swap implementations (API → LocalStorage)
- ✅ Testable với mock repository

### **3.2.4. Infrastructure Layer - Provider (DI)**

```tsx
// providers/AuthProvider.tsx
"use client";

import { useMemo, type ReactNode } from "react";
import { AuthContext } from "../context/AuthContext";
import type { IAuthRepository } from "../interfaces/IAuthRepository";
import { ApiAuthRepository } from "../repositories/ApiAuthRepository";
import { LocalStorageAuthRepository } from "../repositories/LocalStorageAuthRepository";

type AuthRepositoryType = "api" | "localStorage";

interface AuthProviderProps {
  children: ReactNode;
  type?: AuthRepositoryType;
  repository?: IAuthRepository; // For testing
}

/**
 * Auth Provider - Dependency Injection Container
 * Provide repository implementation to the feature
 */
export function AuthProvider({ children, type, repository }: AuthProviderProps) {
  const repositoryInstance = useMemo<IAuthRepository>(() => {
    if (repository) return repository;

    const repoType = type || getDefaultType();
    return repoType === "localStorage"
      ? new LocalStorageAuthRepository()
      : new ApiAuthRepository();
  }, [type, repository]);

  return (
    <AuthContext.Provider value={repositoryInstance}>
      {children}
    </AuthContext.Provider>
  );
}

function getDefaultType(): AuthRepositoryType {
  return process.env.NODE_ENV === "production" ? "api" : "localStorage";
}

```

**✨ Lợi ích:**

- ✅ Centralized dependency wiring
- ✅ Runtime configuration (api vs localStorage)
- ✅ Easy testing (inject mock repository)

### **3.2.5. Application Layer - Hooks**

```tsx
// hooks/useAuthRepository.ts
"use client";

import { useContext } from "react";
import { AuthContext } from "../context/AuthContext";
import type { IAuthRepository } from "../interfaces/IAuthRepository";

/**
 * Hook to access AuthRepository from Context
 * Application layer sử dụng Infrastructure qua interface
 */
export function useAuthRepository(): IAuthRepository {
  const context = useContext(AuthContext);

  if (!context) {
    throw new Error("useAuthRepository must be used within AuthProvider");
  }

  return context;
}

```

```tsx
// hooks/UseLogin.ts
"use client";

import { useMutation } from "@tanstack/react-query";
import { LoginInput } from "../dto/LoginTypes";
import { AuthSession } from "../entities/AuthSession";
import { useAuthRepository } from "./useAuthRepository";

/**
 * Login Hook - Application Business Logic
 * Orchestrate use case flow
 */
export function useLogin() {
  const authRepository = useAuthRepository();

  const mutation = useMutation<AuthSession, Error, LoginInput>({
    mutationFn: async (input: LoginInput) => {
      return await authRepository.login(input);
    },
  });

  return mutation;
}

```

**✨ Lợi ích:**

- ✅ Reusable application logic
- ✅ Access infrastructure via interface
- ✅ Type-safe với generics

### **3.2.6. Presentation Layer - Components**

```tsx
// components/LoginForm.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { useLogin } from "../hooks/UseLogin";
import { validateLogin } from "../usecases/LoginLogic";

/**
 * LoginForm Component - Pure UI
 * Chỉ lo rendering và user interaction
 */
export function LoginForm() {
  const router = useRouter();
  const { mutate: login, isPending, error } = useLogin();

  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [validationError, setValidationError] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setValidationError("");

    // Validate at UI level
    const validation = validateLogin({ email, password });
    if (!validation.success) {
      setValidationError(validation.error);
      return;
    }

    // Execute business logic via hook
    login(validation.data, {
      onSuccess: (session) => {
        session.save();
        router.push("/dashboard");
      },
      onError: (err) => {
        setValidationError(err.message);
      },
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {validationError && (
        <div className="text-red-500 text-sm">{validationError}</div>
      )}

      <Input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        disabled={isPending}
      />

      <Input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        disabled={isPending}
      />

      <Button type="submit" disabled={isPending}>
        {isPending ? "Logging in..." : "Login"}
      </Button>
    </form>
  );
}

```

**✨ Lợi ích:**

- ✅ Component chỉ lo UI/UX
- ✅ Business logic trong hooks
- ✅ Validation ở UI level
- ✅ Dễ test (mock hooks)

### **3.2.7. Entry Point - Page**

```tsx
// page.tsx
import { AuthProvider } from "./providers/AuthProvider";
import { LoginForm } from "./components/LoginForm";

/**
 * Login Page - Feature Entry Point
 * Setup providers và render UI
 */
export default function LoginPage() {
  return (
    <AuthProvider type="localStorage">
      <div className="min-h-screen flex items-center justify-center">
        <LoginForm />
      </div>
    </AuthProvider>
  );
}

```

## 4. Kết luận

Kiến trúc **Feature-Based kết hợp Clean Architecture** mang lại nhiều lợi ích rõ ràng cho các ứng dụng React hiện đại, đặc biệt là các dự án quy mô trung bình và lớn.

- **Lợi ích chính:**
    - **Learning Curve:** Mức trung bình lúc đầu nhưng nhanh chóng giảm nhờ feature isolation và dependency rõ ràng.
    - **Maintenance Cost:** Thấp, dễ refactor, kiểm soát technical debt hiệu quả.
    - **Development / Total Cost:** Thấp, tăng tốc độ thêm feature, fix bug và viết test.
    - **Testability và Reusability:** Cao, domain thuần, usecase và entity dễ tái sử dụng.
    - **Scalability:** Cao, dễ mở rộng module hoặc deploy độc lập, hỗ trợ parallel development và giảm xung đột team.
- **Nhược điểm cần lưu ý:**
    - **Initial Setup Complexity:** Ban đầu cần sắp xếp folder, context và dependency, có thể tốn thời gian chuẩn bị.
    - **Learning Curve:** Thành viên mới cần 1-2 ngày làm quen với tổ chức feature và context pattern.
    - **Overhead cho dự án nhỏ:** Với dự án đơn giản, cấu trúc này có thể hơi “over-engineered”, nhưng vẫn giữ kỷ luật code và dễ bảo trì nếu dự án mở rộng sau này.

**Tóm lại:**

Feature-Based + Clean Architecture là một **giải pháp cân bằng** giữa khả năng mở rộng, maintainability, testability và tốc độ phát triển dài hạn. Mặc dù có chi phí thiết lập ban đầu và đường cong học tập vừa phải, nhưng về lâu dài, kiến trúc này giúp giảm **chi phí bảo trì**, **technical debt**, và tăng **tính ổn định, khả năng mở rộng, và năng suất nhóm**.

---